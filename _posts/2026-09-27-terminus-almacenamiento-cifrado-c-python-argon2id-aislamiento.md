---
title: "Terminus: Almacenamiento Cifrado con C, Python, Argon2id y Aislamiento por Usuario"
date: 2026-09-27 12:00:00 -0600
categories: [Sistemas, Criptografia]
tags: [c, python, libsodium, sqlite, seguridad, almacenamiento, homelab]
toc: true
math: true
image:
  path: /assets/img/posts/terminus-storage-cover.jpg
  alt: "Disco duro mecánico sobre superficie neutra con sus componentes magnéticos expuestos — Fotografía por Denny Müller en Unsplash"
---

> *Fotografía de portada: [Denny Müller](https://unsplash.com/@redaquamedia) en [Unsplash](https://unsplash.com/photos/1qL31aacAPA) (Licencia Unsplash).*
{: .prompt-info}

Cuando publiqué la página de presentación de este blog, escribí una frase muy clara: *"Nació como una forma de documentar mis proyectos personales, como Terminus, y de solidificar mi propio conocimiento"*. Sin embargo, pasaron meses y nunca publiqué el análisis técnico detallado de cómo funciona Terminus por dentro. 

Terminus es mi sistema de almacenamiento conectado directamente (DAS) y micro-almacenamiento en red privado, diseñado desde cero para ejecutarse en hardware modesto de homelab. Su objetivo es garantizar confidencialidad absoluta mediante cifrado en reposo a nivel de byte, sin delegar la seguridad en la capa del sistema de archivos del sistema operativo anfitrión.

Recientemente desarrollé la rama `feat/users-auth`, transformando el prototipo inicial de token estático en una plataforma multi-usuario real con base de datos SQLite embebida, hash de contraseñas con Argon2id integrado en C, aislamiento físico de directorios y protección multicapa contra ataques de *path traversal*. En este artículo detallo cada decisión arquitectónica, el código crítico y las métricas de rendimiento del sistema.

---

## 1. Arquitectura Híbrida: C y Python

La arquitectura de Terminus se divide en dos planos con responsabilidades desacopladas: el motor de cómputo criptográfico y red de bajo nivel escrito en C, y la capa de control, autenticación y enrutamiento en Python.

![Diagrama de Arquitectura de Terminus](/assets/img/posts/terminus-arquitectura-das.svg)

### ¿Por qué C para el núcleo?

En un servidor doméstico o dispositivo embebido, cifrar flujos binarios continuos en Python puro o intermediar buffers en memoria genera una sobrecarga inaceptable por el recolector de basura y el Global Interpreter Lock (GIL). 

Para el motor de red y criptografía seleccioné dos bibliotecas estándar de la industria en C:
1. **GNU libmicrohttpd (`MHD`):** Un servidor HTTP embebible en C, eficiente, ligero y capaz de procesar transferencias binarias mediante sockets directos con baja huella de memoria.
2. **libsodium:** Biblioteca criptográfica moderna y auditada que proporciona primitivas seguras contra ataques de canal lateral (side-channel attacks).

La comunicación entre el proceso C y la lógica de aplicación en Python la estructuré mediante `cffi` en modo `dlopen`, compilando el núcleo C en una biblioteca compartida dinámica: `libterminus_core.so`.

### Cifrado Autenticado en Reposo (AEAD)

Cada archivo que entra en el servidor se cifra antes de tocar el disco magnético o la memoria flash. No almaceno texto plano en ningún momento del ciclo de vida del almacenamiento.

Utilizo la construcción `crypto_secretbox_easy` de libsodium. Esta primitiva implementa cifrado autenticado simétrico combinando el cifrador de flujo XSalsa20 con el código de autenticación de mensajes (MAC) Poly1305:

$$C = \text{XSalsa20}(K, N, M)$$

$$\text{Tag} = \text{Poly1305}(K_{\text{mac}}, C)$$

Donde:
* $K$ es la clave simétrica maestra de 256 bits ($32$ bytes), generada en el primer arranque mediante `os.urandom()` y almacenada con permisos restringidos.
* $N$ es un número de un solo uso (*nonce*) de 192 bits ($24$ bytes) generado aleatoriamente para cada archivo mediante `randombytes_buf()`.
* $M$ es el contenido binario del archivo original.
* $C$ es el texto cifrado resultante.
* $\text{Tag}$ es la etiqueta de autenticación de 128 bits ($16$ bytes).

La función C en `src/c_core/crypto.c` empaqueta el nonce al inicio del payload cifrado:

```c
BytesBuffer encrypt_message(const unsigned char* message, size_t message_len, const unsigned char* key){
  if(sodium_init() < 0) {
    return (BytesBuffer){NULL, 0};
  }

  unsigned long long ciphertext_len = crypto_secretbox_MACBYTES + message_len;
  unsigned char* ciphertext = malloc(ciphertext_len);
  if (ciphertext == NULL){
    return (BytesBuffer){NULL, 0};
  }
  
  unsigned char nonce[crypto_secretbox_NONCEBYTES];
  randombytes_buf(nonce, sizeof(nonce));

  crypto_secretbox_easy(ciphertext, message, message_len, nonce, key);

  size_t total_len = sizeof(nonce) + ciphertext_len;
  char* full_payload = malloc(total_len);
  if (full_payload == NULL){
    free(ciphertext);
    return (BytesBuffer){NULL, 0};
  }

  memcpy(full_payload, nonce, sizeof(nonce));
  memcpy(full_payload + sizeof(nonce), ciphertext, ciphertext_len);
  
  free(ciphertext);

  return (BytesBuffer){full_payload, total_len};
}
```

Al descifrar, la función `decrypt_message` extrae los primeros 24 bytes para recuperar el nonce, valida la etiqueta Poly1305 en tiempo constante y devuelve el contenido original. Si un solo bit del archivo en disco se corrompe o es alterado maliciosamente, `crypto_secretbox_open_easy` falla inmediatamente y la lectura se rechaza.

---

## 2. La Rama `feat/users-auth`: Base de Datos y Credenciales

La primera versión de Terminus dependía de una única variable de entorno `API_SECRET_TOKEN`. Cualquier cliente con esa clave tenía control total. Para convertirlo en un servicio multi-inquilino seguro, diseñé una capa de persistencia transaccional y autenticación por credenciales.

### Hash de Contraseñas con Argon2id en C

El algoritmo elegido para derivar claves de contraseñas es **Argon2id** (RFC 9106), el ganador del Password Hashing Competition. Argon2id combina resistencia contra ataques de canal lateral (Argon2i) y resistencia contra ataques paralelos masivos en GPUs o ASICs (Argon2d).

En lugar de utilizar wrappers lentos en Python, expuse la llamada nativa de libsodium desde C hacia Python:

```c
BytesBuffer hash_password(const char* password, size_t password_len ){
  if(sodium_init() < 0 ){
    return (BytesBuffer){NULL, 0};
  }
  if (password == NULL || password_len == 0) {
    return (BytesBuffer){NULL, 0};
  }
  char* encoded = malloc(crypto_pwhash_STRBYTES);
  if (encoded == NULL) {
    return (BytesBuffer){NULL, 0};
  }
  if (crypto_pwhash_str(encoded, password, password_len, 
                        crypto_pwhash_OPSLIMIT_INTERACTIVE, 
                        crypto_pwhash_MEMLIMIT_INTERACTIVE) != 0) {
    free(encoded);
    return (BytesBuffer){NULL, 0};
  }
  return (BytesBuffer){encoded, strlen(encoded)};
}
```

Configuré el límite interactivo estándar:
* **Operaciones (`OPSLIMIT_INTERACTIVE`):** $2$ iteraciones computacionales.
* **Memoria (`MEMLIMIT_INTERACTIVE`):** $67,108,864$ bytes ($64\text{ MB}$ de memoria RAM dedicada por hash).

Este costo de memoria hace que el ataque por fuerza bruta sobre hashes filtrados sea inviable económicamente para un atacante con hardware acelerador.

### Persistencia Transaccional con SQLite en Modo WAL

Para gestionar las cuentas de usuario creé el módulo `UserStore` en `src/python_wrapper/db.py`. El esquema almacena usuarios, hashes de contraseña y hashes SHA-256 de los tokens de sesión:

```sql
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    api_token_hash TEXT UNIQUE NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_users_username ON users(username);
CREATE INDEX IF NOT EXISTS idx_users_token_hash ON users(api_token_hash);
```

Para asegurar alta concurrencia y tolerancia a fallos en discos lentos, aplico pragmas estrictos al abrir la conexión:

```python
self.conn = sqlite3.connect(db_path, check_same_thread=False)
self.conn.row_factory = sqlite3.Row
self.conn.execute("PRAGMA journal_mode = WAL;")
self.conn.execute("PRAGMA synchronous = NORMAL;")
self.conn.execute("PRAGMA busy_timeout = 5000;")
self.conn.execute("PRAGMA temp_store = MEMORY;")
self.conn.execute("PRAGMA cache_size = -2000;")
```

* **`journal_mode = WAL`:** Permite que los lectores no bloqueen a los escritores y que un escritor no bloquee las consultas de lectura de metadatos.
* **`synchronous = NORMAL`:** Sincroniza el journal en puntos críticos sin emitir `fsync` en cada commit individual, reduciendo la latencia de disco a la mitad sin riesgo de corrupción de base de datos en modo WAL.
* **`busy_timeout = 5000`:** Espera hasta $5$ segundos ante contención antes de arrojar un error de base de datos bloqueada.

### Ciclo de Autenticación de Endpoints

Implementé tres rutas públicas para el ciclo de vida de usuarios:
1. `GET /status`: Monitoreo de salud público que expone la versión del daemon sin requerir credenciales.
2. `POST /auth/register`: Registro de nuevos usuarios con validación de expresiones regulares (`^[a-zA-Z0-9_-]{1,64}$`), longitud de contraseña ($8$ a $128$ caracteres) y límite estricto de payload de $4\text{ KB}$ para prevenir agotamiento de memoria.
3. `POST /auth/login`: Validación de credenciales contra Argon2id y rotación automática del Bearer token usando `secrets.token_urlsafe(32)`.

En la base de datos nunca se guarda el token en texto claro: calculo `hashlib.sha256(token).hexdigest()` para indexar y contrastar las peticiones autenticadas. Para preservar compatibilidad con scripts existentes, mantuve el token legado de administración mediante una comparación en tiempo constante con `secrets.compare_digest()`.

---

## 3. Aislamiento Físico y Blindaje contra Path Traversal

Un error clásico en servidores de almacenamiento es la separación puramente lógica de archivos. Si todos los archivos residen en la misma carpeta física y el software filtra por identificador en una consulta, cualquier fallo en la sanitización permite a un usuario sobrescribir o descargar información ajena.

### Particionado por Espacio de Nombres en Disco

En Terminus, cada usuario tiene su propio subdirectorio físico aislado: `terminus_storage/u_<id>/`.

```
terminus_storage/
├── .secret_key              # Llave simétrica maestra
├── .gitignore
├── u_0/                     # Administrador legado
│   └── backup.tar.enc
├── u_1/                     # Usuario alice
│   └── documento.pdf
└── u_2/                     # Usuario bob
    └── fotografia.raw
```

Para asegurar una transición sin interrupciones, implementé la función `setup_storage()`. Durante el arranque, inspecciona la raíz de almacenamiento y si encuentra archivos creados antes de la actualización multi-usuario, los migra automáticamente hacia el espacio físico `u_0` sin alterar su estado cifrado.

### Cinco Barreras contra Path Traversal

La resolución de rutas de archivo en `resolve_user_file_path()` implementa defensa en profundidad mediante cinco verificaciones secuenciales:

```python
SAFE_FILENAME_REGEX = re.compile(r"^[a-zA-Z0-9][a-zA-Z0-9._-]{0,254}$")
SYSTEM_RESERVED_NAMES = frozenset({
    ".secret_key",
    ".gitignore",
    "config.json",
    "client_config.json",
    "terminus.db",
    "terminus.log",
})

def resolve_user_file_path(user_id: int, filename: str) -> tuple[Path | None, tuple[str, int, str] | None]:
    if not isinstance(filename, str) or not filename.strip() or "\x00" in filename:
        return None, ("Nombre de archivo invalido.", 400, "INVALID_FILENAME")

    # 1. Validación sintáctica por expresión regular
    if not SAFE_FILENAME_REGEX.fullmatch(filename):
        return None, ("Nombre de archivo invalido. Debe comenzar con caracter alfanumerico y no contener rutas relativas.", 400, "INVALID_FILENAME")

    # 2. Bloqueo de archivos reservados del sistema
    if filename.lower() in SYSTEM_RESERVED_NAMES:
        return None, ("Acceso denegado a archivo reservado del sistema.", 403, "FORBIDDEN_FILE")

    user_dir = get_user_storage_dir(user_id)
    
    # 3. Rechazo preventivo de symlinks
    if os.path.islink(user_dir / filename):
        return None, ("No se permiten enlaces simbolicos.", 403, "SYMLINK_FORBIDDEN")

    # 4. Resolución canónica de ruta
    try:
        resolved = (user_dir / filename).resolve(strict=False)
    except Exception:
        return None, ("Ruta no valida.", 400, "INVALID_PATH")

    # 5. Validación de frontera de directorio (Sandbox)
    if not resolved.is_relative_to(user_dir) or resolved == user_dir:
        return None, ("Intento de escape de directorio detectado.", 400, "INVALID_PATH")

    if resolved.is_symlink():
        return None, ("No se permiten enlaces simbolicos.", 403, "SYMLINK_FORBIDDEN")

    return resolved, None
```

Este esquema neutraliza intentos de escape mediante secuencias como `../../etc/passwd`, codificaciones URL maliciosas (`..%2F`), inyecciones de bytes nulos (`\x00`) o symlinks que apunten fuera de la jaula del usuario.

### Optimización de I/O con `os.scandir`

Para listar archivos en `GET /files`, inicialmente utilizaba `os.listdir()`. Sin embargo, `os.listdir` únicamente devuelve una lista de cadenas de texto con los nombres de archivo. Para filtrar subdirectorios u ocultos, el servidor requería ejecutar llamadas adicionales a `os.path.isfile()` por cada elemento del directorio, provocando una avalancha de llamadas al sistema `stat()` sobre el disco.

Reemplacé esa lógica por un iterador contextual con `os.scandir()`:

```python
with os.scandir(user_dir) as entries:
    filtered_files = [
        entry.name for entry in entries
        if not entry.name.startswith('.') and entry.is_file(follow_symlinks=False)
    ]
```

En sistemas Linux, `os.scandir` aprovecha directamente las estructuras `struct dirent` provistas por la llamada al sistema `getdents64`. La información del tipo de archivo (si es fichero regular o enlace) ya viene incluida en el campo `d_type` del inodo, evitando llamadas a `stat` adicionales para cada entrada. En carpetas con miles de archivos cifrados, esto reduce el tiempo de listado en más de un $60\%$.

---

## 4. Métricas y Pruebas Empíricas

Toda mejora de seguridad carece de valor si no está respaldada por una suite de pruebas automatizadas y mediciones directas.

Construí una suite de 50 pruebas automatizadas con `pytest` que evalúa:
* **Criptografía:** Pruebas de ida y vuelta (*roundtrip*) de cifrado y descifrado en C, validando la integridad del mensaje y la detección de alteraciones en la etiqueta Poly1305.
* **Autenticación:** Derivación y verificación de contraseñas con Argon2id, rechazo de contraseñas fuera de rango ($8$ a $128$ caracteres) y descarte de payloads superiores a $4\text{ KB}$.
* **Base de datos:** Inserción y lectura concurrente en SQLite con hilos simultáneos, asegurando que el modo WAL y el `busy_timeout` de $5$ segundos absorban la contención sin bloqueos.
* **Aislamiento de almacenamiento:** Ciclo de vida completo de archivos aislados entre usuarios distintos que suben ficheros con nombres idénticos, confirmando que no existen colisiones ni filtración cruzada.
* **Seguridad de rutas:** Batería automatizada contra vectores de *path traversal* (`../`, secuencias codificadas `%2F`, caracteres nulos y enlaces simbólicos hacia fuera del sandbox).

La ejecución completa de las 50 pruebas toma $2.03$ segundos en el entorno de desarrollo. Este tiempo incluye el arranque y parada dinámica de instancias del servidor C en puertos efímeros, la derivación computacional real de Argon2id ($64\text{ MB}$ por llamada) y la creación y limpieza de los directorios físicos de prueba en disco.

---

## Conclusiones

La deuda técnica que tenía pendiente con este blog queda saldada. Terminus dejó de ser una idea conceptual para convertirse en un microservicio de almacenamiento seguro, robusto y verificable.

Diseñar este sistema me demostró que no es necesario elegir entre la velocidad del lenguaje C y la ergonomía de desarrollo de Python. Mantener la criptografía y las transferencias de sockets crudos en C mientras Python orquesta el modelo de datos, la seguridad en la capa de transporte y las fronteras de directorios proporciona lo mejor de ambos mundos sin sacrificar control.

Todo el código fuente, la configuración de compilación y la suite de pruebas están disponibles en el repositorio [0gerardo0/terminus](https://github.com/0gerardo0/terminus).

---

## Referencias

* **Daniel J. Bernstein.** *Extending the Salsa20 nonce.* Criptoanálisis y especificación de XSalsa20. [cr.yp.to/snuffle/xsalsa-20110204.pdf](https://cr.yp.to/snuffle/xsalsa-20110204.pdf).
* **Daniel J. Bernstein.** *The Poly1305-AES message-authentication code.* Fast Software Encryption (FSE). [cr.yp.to/mac.html](https://cr.yp.to/mac.html).
* **Frank Denis.** *The libsodium cryptographic library: Secret-key authenticated encryption.* [libsodium.gitbook.io](https://libsodium.gitbook.io/doc/secret-key_cryptography/secretbox).
* **A. Biryukov, D. Dinu, P. Khovratovich.** *Argon2: the memory-hard function for password hashing and other applications.* [RFC 9106, IETF](https://www.rfc-editor.org/rfc/rfc9106.html).
* **SQLite Consortium.** *Write-Ahead Logging (WAL) in SQLite.* [sqlite.org/wal.html](https://www.sqlite.org/wal.html).
* **Christian Grothoff et al.** *GNU libmicrohttpd Reference Manual.* [gnu.org/software/libmicrohttpd](https://www.gnu.org/software/libmicrohttpd/).
* **Michael Kerrisk.** *getdents(2) - Linux manual page.* Documentación de llamadas al sistema del kernel de Linux. [man7.org/linux/man-pages/man2/getdents64.2.html](https://man7.org/linux/man-pages/man2/getdents64.2.html).
* **OWASP Foundation.** *File Path Traversal Prevention Cheat Sheet.* [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/cheatsheets/File_Path_Traversal_Cheat_Sheet.html).
* **Armin Rigo, Maciej Fijalkowski.** *CFFI: C Foreign Function Interface for Python.* [cffi.readthedocs.io](https://cffi.readthedocs.io/en/latest/).
