---
title: "Creando IP INFO BAR: Mi extensión para Gnome"
date: 2025-10-13 22:15:00 -0600
categories: [Desarrollo, Proyectos]
tags: [gnome, gjs, python, linux, sistemas]
toc: true
image:
  path: /assets/img/posts/gnome-extension-baner.jpg
  alt: "Logo de GNOME Extension."
---

Como desarrollador, hay pocas cosas más satisfactorias que identificar una pequeña fricción en tu flujo de trabajo diario y construir la herramienta exacta para eliminarla. De esa necesidad nació **IP-INFO-BAR**, mi primera extensión para el escritorio GNOME.

En este post, quiero desglosar no solo qué hace la extensión, sino también el fascinante mundo técnico que hay detrás, los retos que superé y por qué una arquitectura de dos lenguajes fue la integración bastante buena para aprender y solucionar problemas.

## El Problema: Un Clic de Más

La tarea era simple: necesitaba conocer mi dirección IP pública de forma rápida y frecuente. El método tradicional implicaba abrir un navegador, buscar "cuál es mi IP", esperar a que cargara la página y copiar el resultado, otra es utilizar la terminal, que me gusta pero me parece un poco engorroso. Demasiados pasos para una información tan sencilla. Y me parecia que un poco marginal que en otros entornos de escritorio hubiera mas herramientas para tener una información tan necesaria a la vista. Algo que en Gnome no encontre o no tenia una en mantenimiento. 

**La solución era clara:** una pequeña herramienta en la barra superior de mi escritorio que, con un solo clic, me mostrara mi IP y la copiara al portapapeles. Así nació la idea de IP-INFO-BAR. Tengo que aclarar que mi inspiración viene totalmente de que en otros entorno mas para desarrollo tengan esta funcionalidad.

## La Anatomía de una Extensión GNOME 

Contrario a lo que podría parecer, la estructura básica de una extensión para GNOME es sorprendentemente simple. Se compone principalmente de tres archivos:

1.  **`metadata.json`**: La "cédula de identidad" de la extensión. Contiene el nombre, descripción, autor, versión y, lo más importante, el UUID (Identificador Único Universal) que GNOME usa para gestionarla.
2.  **`extension.js`**: El cerebro. Aquí vive toda la lógica, escrita en GJS (GObject JavaScript). Este archivo se encarga de crear los íconos, los menús y de manejar los eventos (como hacer clic).
3.  **`stylesheet.css`**: La apariencia. Un archivo CSS estándar para dar estilo a los elementos de la interfaz, si es necesario.

El reto no está en la cantidad de archivos, sino en entender el entorno en el que se ejecutan.

> GJS no es el JavaScript de tu navegador. Es JavaScript con funcionalidades exclusivas para Gnome DE, capaz de hablar directamente con las librerías del sistema que componen el escritorio GNOME, como Clutter (para gráficos) y GTK (para widgets).
{: .prompt-info}

## La Arquitectura: ¿Por Qué GJS y Python? 

La primera pregunta que me hice fue: ¿puedo hacer todo en GJS? La respuesta es sí, pero no siempre es la forma más sencilla.

-   **GJS para la Interfaz:** GJS es el rey para todo lo relacionado con la UI. Crear un ícono en la barra de estado y un menú desplegable es su especialidad.
-   **Python para los Datos:** Realiza su servicio para obtener datos del sistema y consultar una API externa para información pública por medio de una libreria preenpaquetada `psutils` y `requests` de facil accesibilidad, por su simplicidad de implementación. 

Decidí dividir el proyecto en dos partes para aprovechar lo mejor de cada mundo:

1.  **El Frontend (GJS):** `extension.js` se encarga de mostrar el lo visual y manejar el clic.
2.  **El Backend (Python):** Un script de Python se encarga de hacer la petición web, obtener la IP e inspeccionar el estado local del sistema y devolverla en un formato simple (JSON).

La comunicación entre ambos es sorprendentemente sencilla: `extension.js` simplemente ejecuta el script de Python como un proceso del sistema y captura su salida.

## Funcionamiento Interno

La extensión opera bajo un principio simple pero robusto: una arquitectura de dos partes que separa la interfaz de usuario de la lógica de obtención de datos.

El proceso completo, desde que haces clic hasta que ves tu IP, sigue una coreografía precisa:
1. **Captura del Evento (GJS):** Todo comienza cuando haces clic en la barra. La parte de la extensión escrita en GJS, que está constantemente escuchando, captura este evento.

2. **La Llamada al Backend (Puente):** GJS ejecuta el script de Python como un subproceso síncrono. Esto significa que la interfaz espera pacientemente a que el script termine su trabajo antes de continuar.

```javascript
// GJS ejecuta el script de Python y espera la respuesta.
const [ok, stdout, stderr, exitCode] = GLib.spawn_command_line_sync(
    'python3 /ruta/script/ip_info.py'
);
```
3. **Obtención de Datos (Python):** El script de Python entra en acción. Su misión es actuar como un pequeño centro de inteligencia de red. Donde el servicio devuelve un objeto JSON detallado que contiene la `IPv4`, `IPv6`, `VPN`, `SSH Outgoin and Incoming` si es que se las interfaces de red se encuentran en el sistema y hacer una petición HTTP para extraer la IP publica. 

4. **Retorno y Renderizado (GJS):** El script de Python no guarda archivos, simplemente imprime el resultado JSON en la "salida estándar". GJS captura esta cadena de texto, la parsea con `JSON.parse()` para convertirla en un objeto JavaScript nativo, y finalmente usa ese objeto para actualizar dinamicamente el menú de la barra superior.

## Los Retos, Herramientas y Lecciones Aprendidas. 

El camino estuvo lleno de pequeños obstáculos por lo menos hasta esta primera versión estable, y que se convirtieron en grandes oportunidades de aprendizaje. Mas alla de la espera de la revisión en la tienda de GNOME, los verdaderos desafíos fueron técnicos.

### Etapa 1: Retos de Desarrollo 

#### La Depuración: Epoca de pura oscuridad hasta que...
Al principio, depurar era un proceso a ciergas. No hay una "consola de desarrollador" tradicional. Mi primer método fue el clasico `log()` para imprimir mensajes en los logs del sistema, que luego leía con el comando: `journalctl -f /usr/bin/gnome-shell`.
Era funciona, pero lento. El verdadero cambio de juego fue cuando descubri la depuración en una **ventana anidada**. Al ejecutar
```bash
dbus-run-session -- gnome-shell --nested --wayland
```
puedes lanzar una sesión completa de GNOME dentro de una ventana en tu escritorio actual. Claro esto desde donde utilizo el servidor de visualización de `Wayland`. Esta funcionalidad permite instalar, desinstalar y romper la extensión a voluntad, con una consola de errores en vivo, sin afectar tu sesión de trabajo principal. Fue un salto de productividad gigantesco.

#### La Migración de una Solución "Chusca"

Mi primera versión dependia de que el binario de Python estuviera en una ruta fija o que hubiera dependecias de binarios del sistema que no todas las distribuciones tienen por defecto. Esto es una mala practica, que no funciona en todos los entornos. La solución fue mitigar a un enfoque más robusto, usando herramientas del sistema para encontrar la ruta del inteprete de Python, asegurando que la extensión sea mucho mas portable. 

El primer paso fue dejar de asumir que las herramientas y las rutas de los binarios estan en todos los sistemas. Con esto en mente mi plan a futuro fue utilizar una libreria de Python robusta para poder tener accesible todas las herramientas de monitoreo del sistema. 

`Psutil` (python system and process utilities) es una libreria para obtener información sobre porcesos en ejecución y utilización del sistema (CPU, memoria, discos, red, sensores).  Y de esta manera es que se solucionó el problema de lo fragil que es depender de binarios del sistemas y sus rutas. 

Siendo que este proceso de refactorización me enseño que la primera solución que funciona rara vez es la mejor.

### Etapa 2: Retos de Gestión del Proyecto

Una vez que el código empezó a funcionar, el desafío pasó a ser cómo gestionarlo de una manera profesional y mantenible.

#### El dolor de `.gitignore`

Un error de novato que cometi fue no configurar mi archivo `.gitignore` correctamente desde el inicio. En un `git push` inicial, subi accidentalmente la carpeta `__pycache__`y otros archivos temporales. Corregir el historial o algunos pasos pasos en el controlador de versiones fue clave, puesto que me obligó a adoptar inmediatamente buenas practicas de commit y una mentalidad de Git Flow o lo que yo le llamo Git Flow Lite siendo una version simplificada del popular modelo de ramas Git Flow, enfocada en mantener una rama `master` siempre estable y desarrollar nuevas funcionalidades en ramas separadas, sinedo una forma para mi de hacer un producto mas profesional. 

El problema más severo fue el contrario: al intentar crear un `.gitignore` mas robusto para mi proyecto, inadvertidamente se excluyeron archivos binarios cruciales que requerian las utilidades del script en python. El resultado que en la rama master cualquiera que intentara hacer un `git clone` tendria un proyecto roto. 

### Etapa 3: Retos de Publicación 

Con la extensión funcionando y el proyecto bien gestionado, llegó la fase final: prepararla para el mundo. 

#### Preparación para publicar: El desafio de internacionalización 

A medida que la extensión maduraba, me di cuenta de un reto fundamental para cualquier software de distribución global: el soporte para múltiples idiomas. La internacionalización (conocida como `i18n`) no es simplemente traducir texto; implica preparar toda la estructura del código para que pueda cargar diferentes cadenas de texto dinámicamente.

Es un proceso fascinante y lo suficientemente complejo como para que planee dedicarle un artículo completo en el futuro, donde exploraré cómo implementar un sistema de traducciones mantenible en una extensión GNOME y puedes revisar este post donde hablo de como utilizar la herramienta de [gettext]({% post_url 2025-10-16-internacionalizacion-de-una-extension-gnome %}).

#### La espera y la Paciencia

Una vez que la extensión funcionó en mi maquina, llego el siguiente reto: **la espera**. Para publicar una extensión, la subes al sitio oficial [GNOME Extensions](https://extensions.gnome.org/) y pasa por un proceso de revisión manual. Este proceso puede tardar días o semanas, y en este punto es el que estoy, una lección de paciencia para cualquiera. ¡La sensación de verla finalmente aprovada y disponible será un gran paso!

## Futuras Mejoras y Roadmap

Con una base robusta y funcional ya establecida, el camino está despejado para hacer de IP INFO BAR una herramienta de red aún más potente. Este es un vistazo a las funcionalidades que estoy explorando para futuras versiones:

* **Mejora Radical de la Interfaz:** Un paso interesante es reemplazar el sistema de clics rotativos por un **menú desplegable completo**. Esto permitirá al usuario ver toda la información de un solo vistazo, haciendo la experiencia mucho más rápida e intuitiva.

* **Contexto y Notificaciones Proactivas:**
    * **Geolocalización:** Añadir información geográfica (ciudad/país) asociada a la IP pública para un contexto inmediato.
    * **Alertas de Cambio de IP:** Implementar notificaciones de escritorio que avisen al usuario en tiempo real cuando su IP pública o de VPN cambie.

* **Más Utilidades y Control del Usuario:**
    * **Historial de Conectividad:** Guardar y mostrar las últimas IPs detectadas para un seguimiento rápido de los cambios.
    * **Personalización Avanzada:** Dar al usuario control total para reordenar u ocultar la información que no necesita.
    * **Generador de QR:** Añadir una opción para generar un código QR con la IP actual, facilitando su transferencia a dispositivos móviles.

## Conclusión 
Crear IP Info Bar fue un viaje de aprendizaje inmenso. No solo solucionó un problema personal, sino que me abrió las puertas al desarrollo de escritorio en Linux de una forma moderna y accesible. Dividir la logica entre GJS y Python demostró se una estrategia robusta que planeo usar en futuros proyectos.

Si te interesa, puedes encontrar la extension en la tienda [tienda oficial de extensiones de GNOME](https://example.com) y revisar todo el codigo fuente en su [repositorio de Github](https://github.com/0gerardo0/IP-Info-Bar).

> Como se menciona en el cuerpo del texto a fecha de que que se escribio este articulo la extensión sigue en proceso de revisión. Cuando este aprovada subire un post mecionando como fue el proceso y las veces que fue rechazada la extensión.
{: .prompt-info}

## Referencias 

* GNOME. (s.f.). Writing Your First Extension. GNOME Developer Documentation. [GJS Guide: Getting Started](https://gjs.guide/extensions/development/creating.html)

* GNOME. (s.f.). GJS Guide. GJS. [GJS Docs: GNOME](https://gjs-docs.gnome.org/)

* GNOME. (s.f.). *Running and Debugging*. GJS Guide. [GJS Debugging](https://gjs.guide/extensions/development/debugging.html)

* Waterman, J. (2024). *psutil* (Version 5.9.8) [Software]. Python Package Index. [psutil PyPi](https://pypi.org/project/psutil/)

* GNOME. (s.f.). *Translations*. GJS Guide. [GJS Development: Translations](https://gjs.guide/extensions/development/translations.html)

* Zúñiga, G. (2025). *IP-Info-Bar* [Software]. GitHub. [IP INFO BAR GitHub](https://github.com/0gerardo0/IP-Info-Bar)

* Atlassian. (s.f.). *Gitflow workflow*. [https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)

* Python Software Foundation. (s.f.). *json — JSON encoder and decoder*. The Python Standard Library. [https://docs.python.org/3/library/json.html](https://docs.python.org/3/library/json.html)
 
