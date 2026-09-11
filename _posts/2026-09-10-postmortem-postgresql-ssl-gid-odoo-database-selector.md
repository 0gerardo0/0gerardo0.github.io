---
title: "Post-Mortem: El GID Huérfano que Tiró PostgreSQL y Expuso el Selector de Bases de Datos de Odoo"
date: 2026-09-10 18:30:00 -0600
categories: [DevOps, SysAdmin]
tags: [odoo, postgresql, debian, linux, nginx, seguridad, sysadmin]
toc: true
image:
  path: /assets/img/posts/postmortem-postgres-ssl-cover.jpg
  alt: "Pieza de rompecabezas oscura suspendida sobre una ranura vacía con luz filtrándose — Fotografía por Edge2Edge Media en Unsplash"
---

> *Fotografía de portada: [Edge2Edge Media](https://unsplash.com/@edge2edgemedia) en [Unsplash](https://unsplash.com/photos/x21KgBfOd_4) (Licencia Unsplash).*
{: .prompt-info}

Un reinicio rutinario de servidor después de una jornada de *hardening* y mantenimiento parecía el cierre perfecto de la noche. Sin embargo, al recargar el navegador para verificar la disponibilidad de mi instancia de Odoo 18 en producción, no me recibió la interfaz habitual del ERP ni la pantalla de login: me recibió un error HTTP 500 y, tras reintentar, la pantalla pública de **selección de bases de datos (`/web/database/selector`)**.

En este post documento la autopsia técnica de la falla: cómo un ajuste de seguridad a nivel de sistema operativo rompió silenciosamente el cluster de PostgreSQL 15, por qué Odoo degradó su comportamiento exponiendo información de esquemas, y las medidas de defensa en profundidad que implementé para que esto no vuelva a ocurrir.

---

## 1. El Incidente: La Cascada de Fallos

El síntoma visible comenzó inmediatamente tras ejecutar un `systemctl reboot` programado en mi servidor con **Debian 12**.

En lugar de cargar el sitio web corporativo mapeado a la base de datos de producción mediante el filtro de host de Nginx, el navegador arrojó:

```
Internal Server Error
The server encountered an internal error and was unable to complete your request.
```

Al inspeccionar los logs de Odoo en `/var/log/odoo/odoo.log`, encontré una ráfaga continua de excepciones de conexión hacia la base de datos:

```text
2026-08-30 02:14:11,102 1240 ERROR ? odoo.sql_db: Connection to the database failed
Traceback (most recent call last):
  File "/opt/odoo/odoo/odoo/sql_db.py", line 640, in connection
    return self._pool.get()
  ...
psycopg2.OperationalError: could not connect to server: No such file or directory
	Is the server running locally and accepting
	connections on Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?
```

Odoo intentaba comunicarse a través del socket Unix local `/var/run/postgresql/.s.PGSQL.5432`, pero el archivo de socket ni siquiera existía. El servicio de base de datos no estaba respondiendo.

---

## 2. Diagnóstico: El Error Criptográfico en PostgreSQL

Me dirigí de inmediato a consultar el estado del servicio mediante `systemctl`:

```bash
sudo systemctl status postgresql@15-main.service
```

La salida confirmó que la unidad de systemd había colapsado durante la fase de inicio:

```text
● postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: failed (Result: exit-code) since Sat 2026-08-30 02:14:05 UTC; 1min ago
    Process: 985 ExecStart=/usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main ... (code=exited, status=1/FAILURE)
```

Para ver el motivo exacto del rechazo, extraje los registros del journal de systemd:

```bash
sudo journalctl -u postgresql@15-main.service -n 25 --no-pager
```

Allí apareció la causa raíz del problema:

```text
2026-08-30 02:14:04.812 UTC [985] FATAL:  could not access private key file "/etc/ssl/private/ssl-cert-snakeoil.key": Permission denied
2026-08-30 02:14:04.814 UTC [985] LOG:  database system is shut down
pg_ctl: could not start server
Examine the log output.
```

PostgreSQL se negaba a arrancar porque **no podía leer la llave privada SSL**.

---

## 3. Causa Raíz: El GID Huérfano tras el Hardening

¿Por qué el usuario del sistema `postgres` perdió repentinamente acceso a un certificado que había funcionado durante meses?

En Debian y Ubuntu, los certificados autofirmados del sistema se gestionan mediante el paquete `ssl-cert`. Por diseño de seguridad POSIX, el directorio `/etc/ssl/private/` tiene permisos estrictos `710` (`drwx--x---`), con propietario `root` y grupo `ssl-cert`:

```
Directorio: /etc/ssl/private
Permisos:   drwx--x--- (710)
Owner:      root
Group:      ssl-cert
```

Cualquier demonio que necesite leer certificados SSL (como PostgreSQL o Exim) debe tener a su usuario asignado como miembro del grupo complementario `ssl-cert`. En mi servidor, el usuario `postgres` efectivamente pertenecía a dicho grupo:

```bash
$ id postgres
uid=107(postgres) gid=115(postgres) groups=115(postgres),112(ssl-cert)
```

El problema se originó horas antes durante una sesión de auditoría y remediación de seguridad con **Lynis**. Como parte del saneamiento de cuentas de sistema y reasignación de rangos numéricos de IDs, se recreó el grupo `ssl-cert` asignándole el nuevo **GID 112**.

Sin embargo, los inodos en disco en el sistema de archivos ext4 almacenan los identificadores numéricos puros (GID), no los nombres de texto. Al consultar los permisos numéricos con `ls -ldn`:

```bash
$ ls -ldn /etc/ssl/private
drwx--x--- 2 0 104 4096 Aug 12 19:30 /etc/ssl/private
```

El directorio en disco seguía apuntando al **GID huérfano 104**.

```
[ Inodo en Disco ]                  [ Memoria del Kernel /etc/group ]
/etc/ssl/private  ───────► GID: 104 (Huérfano, ya no existe)
Usuario postgres  ───────► GID: 112 (Nuevo ID de ssl-cert)
                                ▲
                                │
                   ¡DISCREPANCIA DE IDENTIFICADOR!
                   Kernel POSIX rechaza con EACCES (13)
```

Cuando PostgreSQL intentó abrir `/etc/ssl/private/ssl-cert-snakeoil.key`, el kernel evaluó:
1. ¿El UID `107` es `0` (root)? **No.**
2. ¿El GID primario `115` o suplementario `112` coincide con el GID `104` del directorio? **No.**
3. ¿Tienen permiso los *otros* (`other`)? **No (`---`).**

Resultado: **`EACCES: Permission denied`**. Al abortar el arranque de PostgreSQL, no se creó el socket Unix, colapsando el ERP completo.

---

## 4. El Efecto Secundario: ¿Por qué Odoo expuso el Database Selector?

El aspecto más delicado de este incidente no fue la caída de la base de datos, sino **cómo reaccionó Odoo ante el fallo**.

Cuando Odoo recibe una petición HTTP a través del proxy inverso (Nginx), su despachador ejecuta la lógica de resolución de tenant mediante la directiva `dbfilter`. Si la conexión con PostgreSQL está viva, Odoo consulta la lista de esquemas, evalúa la expresión regular contra el encabezado `Host` y sirve la base de datos correspondiente.

Sin embargo, cuando el socket de base de datos no responde en absoluto:
1. El ORM captura la excepción `OperationalError` al no poder consultar `pg_database`.
2. Al no poder validar la existencia de ninguna base de datos configurada, el controlador web asume que se trata de una instancia en fase de inicialización o con bases de datos no seleccionadas.
3. Si el parámetro `list_db` en `/etc/odoo.conf` está configurado en `True` (o no está definido, ya que `True` es el valor por defecto), la ruta `/web` redirige automáticamente a `/web/database/selector`.

Aquí es donde entró en juego una **falsa sensación de seguridad** en mi infraestructura. En mi configuración de Nginx yo ya contaba con una regla de bloqueo:

```nginx
# La regla que tenía en Nginx antes del incidente:
location ~ /web/database/manager {
    deny all;
    return 404;
}
```

Yo creía tener el perímetro cubierto porque `/web/database/manager` devolvía un 404. Pero Odoo separa la creación/respaldo (`manager`) de la selección de esquemas (`selector`). Al no tener `list_db = False` en `odoo.conf` y al no filtrar la palabra `selector` en Nginx, el fallo de PostgreSQL abrió las puertas de par en par a la interfaz de selección.

> **Riesgo OPSEC:** En un servidor de producción accesible desde Internet, exponer el selector de bases de datos revela los nombres exactos de los esquemas corporativos, versiones instaladas y ofrece una superficie de ataque para fuerza bruta si no se cuenta con una contraseña maestra (*master password*) robusta.
{: .prompt-warning}

---

## 5. La Solución Paso a Paso

### Paso 1: Corregir la propiedad del grupo en el sistema de archivos
Reasigné el grupo `ssl-cert` al directorio y a las llaves privadas correspondientes para sincronizar el inodo con el nuevo GID:

```bash
# Reasignar el grupo correcto al directorio privado
sudo chgrp -R ssl-cert /etc/ssl/private

# Asegurar permisos POSIX estrictos (710 en directorio, 640 en llaves)
sudo chmod 710 /etc/ssl/private
sudo chmod 640 /etc/ssl/private/*
```

Verifiqué la corrección numérica:

```bash
$ ls -ldn /etc/ssl/private
drwx--x--- 2 0 112 4096 Aug 30 02:22 /etc/ssl/private
```

Ahora el GID del directorio (`112`) coincide exactamente con el GID suplementario del usuario `postgres` (`112`).

### Paso 2: Reiniciar el cluster de PostgreSQL y verificar el socket
Reinicié el servicio de base de datos y comprobé la creación inmediata del socket Unix:

```bash
sudo systemctl restart postgresql@15-main.service
sudo systemctl status postgresql@15-main.service
```

Confirmé la presencia del socket activo:

```bash
$ ls -l /var/run/postgresql/.s.PGSQL.5432
srwxrwxrwx 1 postgres postgres 0 Aug 30 02:23 /var/run/postgresql/.s.PGSQL.5432
```

### Paso 3: Reiniciar Odoo y validar el tráfico
Con PostgreSQL levantado y respondiendo consultas en menos de 2 milisegundos, reinicié el servicio de Odoo:

```bash
sudo systemctl restart odoo.service
```

Al recargar el dominio corporativo, la página respondió con `HTTP 200 OK` directo hacia el portal web, eliminando cualquier pantalla intermedia.

---

## 6. Blindaje y Defensa en Profundidad

Para asegurar que ni este ni ningún otro fallo futuro en el motor de base de datos vuelva a exponer las rutas de Odoo, apliqué dos capas de blindaje permanente en mis plantillas de infraestructura:

### Capa 1: Desactivar `list_db` en Odoo
En el archivo de configuración `/etc/odoo.conf` (y en la plantilla correspondiente de mi playbook de despliegue), aseguré la desactivación explícita del listado de bases de datos:

```ini
[options]
; Desactiva por completo el selector y gestor público de BD
list_db = False
```

Con `list_db = False`, si PostgreSQL cae o no responde al enrutamiento, Odoo arroja un error interno genérico, pero **jamás ofrece la interfaz interactiva para listar o gestionar esquemas**.

### Capa 2: Bloqueo perimetral integral en Nginx
Modifiqué la directiva de Nginx en los VirtualHosts para extender la denegación tanto a `manager` como a `selector`:

```nginx
# Bloquear acceso público tanto al manager como al selector
location ~* ^/web/database/(manager|selector) {
    deny all;
    return 404;
}
```

Al devolver un `404 Not Found` en el proxy inverso antes de que la petición toque el servidor WSGI de Odoo en el puerto 8069, la ruta queda completamente neutralizada del perímetro público.

---

## Lecciones Aprendidas

1. **El software de hardening automatizado requiere validación de inodos:** Cambiar GIDs o UIDs en `/etc/group` o `/etc/passwd` no actualiza mágicamente los archivos en disco. Tras cualquier reasignación de IDs, es imperativo correr una auditoría de archivos huérfanos:
   ```bash
   # Buscar archivos en el sistema cuyo GID no pertenezca a ningún grupo actual
   sudo find /etc /var /opt -nogroup -o -nouser
   ```
2. **PostgreSQL es estricto por diseño con SSL:** A diferencia de otros demonios que emiten advertencias y continúan, PostgreSQL abortará el arranque si tiene configurado `ssl = on` y no puede verificar la propiedad y permisos de la llave privada.
3. **Nunca confíes en el comportamiento por defecto de un ERP ante fallos:** Los frameworks web a menudo tienen mecanismos de conveniencia para desarrollo (como el database selector automático) que se convierten en vectores de fuga de información en producción. La directiva `list_db = False` debe ser obligatoria desde el día cero.

---

## Conclusiones

Los incidentes de infraestructura más desconcertantes rara vez se deben a bugs complejos en el código de la aplicación; casi siempre nacen en las intersecciones invisibles entre el sistema operativo, los permisos de bajo nivel y las asunciones por defecto del software.

Este post-mortem dejó en evidencia que las tareas de hardening automatizado no terminan cuando el script de auditoría devuelve un score alto. La seguridad real reside en la coherencia entre el espacio de usuario, los inodos del sistema de archivos y las políticas de degradación elegante de cada servicio ante caídas catastróficas.

Ambas correcciones —la auditoría de inodos para llaves SSL y el endurecimiento de Nginx y Odoo para silenciar definitivamente el selector de bases de datos— quedaron formalmente integradas y versionadas en mi playbook de aprovisionamiento ([`odoo-server-playbook`](https://github.com/0gerardo0/odoo-server-playbook)), asegurando que cualquier despliegue o nodo futuro nazca blindado por diseño.

---

## Referencias

* **PostgreSQL Global Development Group (2023).** *Secure TCP/IP Connections with SSL — Server-Side Setup & Key File Security.* Documentación oficial de PostgreSQL 15 sobre la directiva `ssl = on` y validación de permisos en llaves privadas (`0600`/`0640`). [https://www.postgresql.org/docs/15/ssl-tcp.html](https://www.postgresql.org/docs/15/ssl-tcp.html)

* **PostgreSQL Global Development Group (2023).** *Managing Connections — Unix-Domain Sockets (`unix_socket_directories`).* Especificación del ciclo de vida del socket `.s.PGSQL.5432` y control de concurrencia local. [https://www.postgresql.org/docs/15/runtime-config-connection.html](https://www.postgresql.org/docs/15/runtime-config-connection.html)

* **Debian Policy Manual.** *Section 9.2.2: System users and groups (`ssl-cert`).* Especificación de empaquetado y aislamiento de permisos para servicios que requieren acceso a certificados compartidos en Debian GNU/Linux. [https://www.debian.org/doc/debian-policy/ch-opersys.html#system-users-and-groups](https://www.debian.org/doc/debian-policy/ch-opersys.html#system-users-and-groups)

* **Debian Package Tracker.** *Package: ssl-cert & make-ssl-cert(8).* Manual de mantenimiento y aprovisionamiento seguro del directorio `/etc/ssl/private` (modo `710`). [https://manpages.debian.org/bookworm/ssl-cert/make-ssl-cert.8.en.html](https://manpages.debian.org/bookworm/ssl-cert/make-ssl-cert.8.en.html)

* **Debian Project.** *Securing Debian Manual — File permissions and system integrity verification.* Guía oficial de hardening y gestión de permisos en Debian. [https://www.debian.org/doc/manuals/securing-debian-manual/](https://www.debian.org/doc/manuals/securing-debian-manual/)

* **The Open Group / IEEE Std 1003.1-2017 (POSIX.1).** *File Access Permissions & Inode Metadata (`sys/stat.h`, `stat(2)`).* Estándar internacional sobre resolución de bits de permisos y desacoplamiento entre UIDs/GIDs numéricos en disco y nombres en `/etc/group`. [https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/sys_stat.h.html](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/sys_stat.h.html)

* **Linux Kernel Organization.** *inode(7) — Linux manual page on inode metadata and filesystem attributes.* Manual del kernel Linux sobre estructuras de inodos y resolución de identidades numéricas. [https://man7.org/linux/man-pages/man7/inode.7.html](https://man7.org/linux/man-pages/man7/inode.7.html)

* **Odoo S.A.** *Odoo Server Configuration and Command-line Interface (`list_db` and `dbfilter`).* Documentación oficial de despliegue, opciones de arranque y comportamiento multi-inquilino en Odoo. [https://www.odoo.com/documentation/18.0/developer/reference/cli.html](https://www.odoo.com/documentation/18.0/developer/reference/cli.html)

* **Odoo Community Association (OCA).** *dbfilter_from_header — Dynamic database routing via reverse proxy headers.* Repositorio oficial de OCA server-tools para aislamiento perimetral de dominios hacia bases de datos. [https://github.com/OCA/server-tools](https://github.com/OCA/server-tools)

* **Nginx Documentation.** *Module ngx_http_core_module — location directive syntax and access rules (`allow`, `deny`).* Documentación técnica del servidor web y proxy reverso Nginx. [https://nginx.org/en/docs/http/ngx_http_core_module.html](https://nginx.org/en/docs/http/ngx_http_core_module.html)

* **CISOfy.** *Lynis — Security auditing and compliance tool for UNIX systems.* Documentación oficial del motor de auditoría de permisos, cuentas de sistema y hardening. [https://cisofy.com/lynis/](https://cisofy.com/lynis/)

