---
title: "Internacionalización de una extensión GNOME"
date: 2025-10-16 18:42:00 -0600
categories: [Desarrollo, Tutoriales]
tags: [gnome, gjs, i18n, gettext, linux, makefile]
toc: true
image:
    path: ../assets/img/posts/i18n-banner-.jpeg
    alt: "Banner de gato con letras"
---

Este post es una guía práctica sobre el proceso de traducción de software. Quiero compartir mi entendimiento sobre lo que conlleva la internacionalización, ya que me pareció interesante aprender sobre esta utilidad y lo potente que puede ser. Exploraremos el sistema `gettext` , cómo automatizar el proceso con un `Makefile`  y las lecciones que aprendí al preparar mi extensión para ser multilingüe.

En mi post sobre la [creación de IP INFO BAR]({% post_url 2025-10-13-Creando-IP-INFO-BAR-Mi-Extensión-para-GNOME %}), mencioné que uno de los grandes retos para que este proyecto de software madure es la **internacionalización (i18n)**. Una vez que pude presentar la extensión como funcional, el siguiente paso era hacerla accesible para otros usuarios que no hablen mi idioma 

## Traducción 'gettext'

La internacionalización en el ecosistema GNOME (Y en gran parte del mundo del software libre) se basa en esta herramienta: **GNU `gettext`**.

La forma en la que puedo pensar que funciona este sistema es como un diccionario para la aplicación. La se puede simplificar asi:

1. **Marcar el texto:** En el codigo se marcan todas las cadenas de texto que uno como desarrollador pone para describir etiquetas, mensajes o cualquier cadena que se quiera traducir.

2. **Extraer el texto:** Donde se coloca la funcion de `gettext` es donde escanea el codigo, y extrae todas las cadenas marcadas y las pone en un archivo de plantilla.

3. **Traduces:** La traducción consiste en tener como base la plantilla y hacer una copia para cada idioma que se quiera soportar para la aplicación, aunque es tedioso para aplicaiones extensas, se rellena la plantilla para la traducción. 

### Los archivos clave 

* **`.pot` (Portable Object Template):** La plantilla principal. Contiene todas las cadenas de texto extraidas del proyecto, texto sin traducción.
* **`.po` (Portable Object):** El archivo de trabajo. Es una copia de la plantilla `.pot` para un idioma especifico (ej. `es.po`, `ru.po`,  etc.) donde se ecriben las trducciones en texto plano. Bastante intuitivo y facil de editar.
* **`.mo` (Machine Object):** La versión compilada y binaria de `.po`. Es un archivo que la aplicación utilizara en producción para buscar las trducciones.

## Paso 1: Marcar el texto en el Código

El primer paso es decirle a `gettext` qué cadenas de texto se necesitan traducir. Esto se hace con una función especial, que por conveniencia se renombre por un guion bajo: `_()`.

Primero, es importar la herramienta en la extension `extension.js`:

```javascript
  import {Extension, gettext as _} from 'resource:///org/gnome/shell/extensions/extension.js'
```

Luego, en cualquier lugar en donde se define un texto para la interfaz, se aplica la función especial `_()`:

```javascript
  // Antes:
  let label = 'IP Address';

  // Después, envuelto en la función de traducción:
  let label = _('IP Address');
```
La función `_()` actúa como un marcador que `gettext` puede encontrar. En tiempo de ejecución, esta misma función se encargará de buscar la traducción correcta para "IP Address" en el idioma del usuario.

## Paso 2: El Flujo de Trabajo: Manual vs. Automatizado

Extraer, actualizar y compilar los archivos de traducción siempre es tedioso y se cometen algunos errores, o es propenso a fallar. Lo mejor para automatizar esta tarea es utilizar un script `Makefile`, donde se convierte en archivo que puede hacer este conjunto de tareas.

Para entender realmente el poder de un `Makefile`, primero veamos cómo seria el proceso de traducción si se hace a mano desde la terminal.

### El proceso manual:

Para entender como es que va a funcionar la automatización, primero se necesita entender cual es la funcion de estos comandos.

### Extraer las cadenas de Texto (`xgettext`)

Con el comando con unos parametros especificos es como se extraen las cadenas de todo el código fuente (`.js`), que esta marcado con la función especial `_()` y poder tener una plantilla con la extensión .pot.

```bash
    xgettext --from-code=UTF-8 -o po/ip-info-bar.pot --language=JavaScript --keyword=_ extension.js prefs.js
```

Donde `xgettext` es la herramienta para escanear el codigo, el parametro -o especifica la salida de la que sera la plantilla, el parametro de lenguaje indica con con cual se esta trabajando y finalmente el parametro `--keyword` que le indica al comando que funcion es la marca donde hay una cadena traducible.

### Compilar las Traducciones (`msgfmt`)

Con el archivo con las traducciones listas en la copia del idioma al que se traduce (ej. `es.po`), se necesita compilar al formato binario `.mo` que la extension pueda usar. 

```bash
    mkdir -p locale/es/LC_MESSAGES/

    msgfmt po/es.po -o locale/es/LC_MESSAGES/ip-info-bar.mo
```

El comando lo que hace es primero asegurar que la estructura de carpetas exista, y luego se compila el archivo. Donde `msgfmt` es la herramienta que compila el archivo `.po` y los siguiente parametros indican la entrada y salida de archivos. Algo que se tiene que mencionar es que la ruta de los archivos no es arbitraria, es lo que espera `gettext` como estructura de directorios para encontrar y cargar las trducciones (`locale/<idioma>/LC_MESSAGES/`).

 ### El proceso Automatizado:

Como se puede ver, los comandos son engorrosos y se pueden cometer errores, es facil la implementación pero siempre se puede hacer más facil. Además, tendriamos que repetirlo por cada idioma y cada vez que se actualiza el codigo. 

### La Automatización con un `Makefile`

Esta es la genialidad de estos archivos, para evitar hacer ese proceso manual y crear un flujo de trabajo, se automatiza con un `Makefile`, este es un script que ejecuta estos mismos comandos por nosotros.  

Como ejemplo, el script que se utilizo en IP INFO BAR: 

```bash
GETTEXT_PACKAGE = ip-info-bar
I18N_DIR = po

POT_FILE = $(I18N_DIR)/$(GETTEXT_PACKAGE).pot
PO_FILES = $(wildcard $(I18N_DIR)/*.po)

update-pot:
	xgettext --from-code=UTF-8 -o $(POT_FILE) --language=JavaScript --keyword=_ extension.js prefs.js

update-po: update-pot
	for lang in $(PO_FILES); do \
		msgmerge -U $$lang $(POT_FILE); \
	done

compile-mo:
	for lang in $(PO_FILES); do \
		lang_code=$$(basename $$lang .po); \
		mkdir -p locale/$$lang_code/LC_MESSAGES; \
		msgfmt $$lang -o locale/$$lang_code/LC_MESSAGES/$(GETTEXT_PACKAGE).mo; \
	done
```

Para simplificar la explicación de este código,se divide en tres comandos el flujo de trabajo:

1. `make update-pot` para generar la plantilla.

2. `make update-po` para actualizar los archivos de traducciones.

3. `make compile-mo` compila todo para producción.

## Conclusión 

Aunque la internacionalización puede parecer un paso secundario, es fundamental para que un proyecto de software alcance una audiencia global, ya que esto hace que la comunidad pueda colaborar, revisar y mejorar las traducciones. Estas herramientas como `gettext` y la automatización `Makefile` la convierte en un proceso manejable y estructurado. Y preprar mi extensión IP INFO BAR para ser multilingüe fue un buen aprendizaje para la contruccion de software y pueda tener un alcance global.

Es algo interesante como esta es una de las piezas para la introducción colaboración en los proyectos de codigo abierto. 

## Referencias

* The GNOME Project. (s.f.). *Translations*. GJS Guide. [GJS Guide: Translations](https://gjs.guide/extensions/development/translations.html)

* GNU Project. (2024). *GNU `gettext`utilities*. [GNU: gettext](https://www.gnu.org/software/gettext/)
