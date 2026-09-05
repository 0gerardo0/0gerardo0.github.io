---
title: "SEO Técnico con Nginx y Odoo: Sitemap Limpio, Landing Pages y Desindexado de Staging"
date: 2026-09-05 06:30:00 -0600
categories: [Infraestructura, DevOps]
tags: [seo, odoo, nginx, sitemap, robots, cloudflare, landing-pages, open-source]
toc: true
image:
  path: /assets/img/posts/seo-nginx-odoo-banner.png
  alt: "Esquema de SEO técnico: Odoo detrás de Nginx sirviendo sitemap y robots."
---

Cuando usas un ERP como **Odoo** para montar un sitio web comercial, el SEO no viene regalado. La plataforma genera contenido dinámico, rutas de módulos de demostración y metadatos que, por defecto, no están pensados para destacar en Google. En este post te cuento cómo resolví el problema de raíz: tomé el control del `sitemap.xml`, automaticé los metadatos de 17 landing pages con scripts y separé el *staging* de la producción para no canibalizar mi propio tráfico.

La clave de todo: **Nginx como capa de optimización frente al ERP**, y un puñado de scripts Python hablando con Odoo por XML-RPC.

## El Problema: el Sitemap "Sucio" de Odoo

Odoo Community incluye módulos técnicos de demo (`website_event`, `website_hr_recruitment`, `website_sale`). Son útiles para aprender, pero cuando activas la tienda de un ERP como sitio web, estos módulos **cuelan rutas basura en el sitemap dinámico** que genera la plataforma:

```
/events
/jobs
/shop
/groups
```

Peor aún, a veces aparecen con esquema `http://` en vez de `https://`. El resultado es un `sitemap.xml` lleno de URLs que no quieres indexar y que diluyen el peso de las páginas que sí te importan.

**La lección:** el sitemap dinámico de un framework casi nunca es tu mejor carta de presentación ante un buscador. Cuando tu página depende de él para el rastreo, ese archivo debería ser tuyo, no del generador.

## Sitemap XML Estático Servido por Nginx

La solución práctica fue dejar de depender del sitemap de Odoo y **servir un `sitemap.xml` estático directamente desde Nginx**. Así tengo control absoluto sobre las URLs indexadas: HTTPS puras, sin rutas de demo y con tiempos de respuesta del orden de **~2 ms** porque Nginx no consulta a la aplicación.

```nginx
location = /sitemap.xml {
    root /var/www/mi-sitio;
    default_type application/xml;
    add_header Content-Type "application/xml; charset=utf-8";
    expires 1d;
}
```

Dentro de `/var/www/mi-sitio/sitemap.xml` defino solo las URLs comerciales que quiero indexar (en mi caso, 24 URLs HTTPS puras). La respuesta se cachea un día para no golpear el origen.

Verificación rápida del header:

```bash
curl -sIL https://<tu-dominio-prod>/sitemap.xml | grep -E "^HTTP|content-type"
```

## Las 17 Landing Pages y su Arquitectura en Odoo

En Odoo, cada landing page es la combinación de **dos registros** en la base de datos:

1. **Vista QWeb (`ir.ui.view`)**: la estructura HTML (`arch_db`), con estilos de Bootstrap y el contenido de los bloques.
2. **Registro de Página Web (`website.page`)**: la URL canónica, si está publicada (`is_published`), su visibilidad ante buscadores (`website_indexed`) y las metaetiquetas SEO (`website_meta_title`, `website_meta_description`, `website_meta_keywords`).

Organice el catálogo en tres grupos para que la arquitectura de información fuera coherente:

| Grupo | Ejemplos de rutas | Enfoque |
| :--- | :--- | :--- |
| **Soluciones** | `/facturacion-electronica`, `/cadena-de-suministro` | Capacidades del ERP |
| **Ecosistema** | `/seguridad-y-hardening`, `/respaldos-automaticos` | Infraestructura y servicios |
| **Recursos** | `/preguntas-frecuentes`, `/comparativas-erp` | Soporte y contenido educativo |

Todos los botones de llamado a la acción enlazan a la URL canónica `/contactus?service=<parametro>` para preservar el origen del prospecto en el CRM.

## Jerarquía Semántica Estricta (1 H1, H2, H3)

El SEO técnico no es solo metadatos: también es **estructura del HTML**. Establecí una regla que se repite en todas las páginas:

* **Un solo `<h1>`** en la sección de portada (*Hero*). Describe la propuesta de valor de la página.
* **`<h2>`** para los bloques intermedios ("Funciones clave", "Proceso en 4 pasos", "Llamado a la acción").
* **`<h3>`** para las tarjetas y características individuales dentro de las columnas.

Sin saltos de nivel y sin múltiples H1. Esto mejora la accesibilidad y le da a Google una interpretación clara de la jerarquía del contenido.

## Automatización con Scripts Python (XML-RPC)

Gestionar a mano los metadatos de 17 páginas en la interfaz web es tedioso y propenso a errores. Escribí tres scripts en Python que hablan directamente con Odoo por XML-RPC. Aquí los muestro con valores placeholder; las credenciales reales nunca van en el código fuente de un post.

### 1. Auditar todas las páginas publicadas

Este script lista cada página publicada y marca qué metadatos faltan:

```python
# seo_audit.py
import xmlrpc.client

URL = 'https://<tu-dominio-prod>'
DB = '<tu-db>'
USERNAME = '<tu-usuario>'
PASSWORD = '<tu-password>'

common = xmlrpc.client.ServerProxy(f'{URL}/xmlrpc/2/common')
uid = common.authenticate(DB, USERNAME, PASSWORD, {})
models = xmlrpc.client.ServerProxy(f'{URL}/xmlrpc/2/object')

domain = [('is_published', '=', True)]
pages = models.execute_kw(
    DB, uid, PASSWORD, 'website.page', 'search_read',
    [domain],
    {'fields': ['url', 'name', 'website_meta_title',
                'website_meta_description', 'website_meta_keywords']}
)

for p in pages:
    title = p.get('website_meta_title') or '[MISSING]'
    desc = p.get('website_meta_description') or '[MISSING]'
    print(f"{p['url']} | {title} | {desc}")
```

La salida me permite detectar de inmediato las páginas que quedaron sin título o descripción.

### 2. Escribir metadatos vía XML-RPC

```python
# update_page_seo.py
import xmlrpc.client

URL = 'https://<tu-dominio-prod>'
DB = '<tu-db>'
USERNAME = '<tu-usuario>'
PASSWORD = '<tu-password>'

common = xmlrpc.client.ServerProxy(f'{URL}/xmlrpc/2/common')
uid = common.authenticate(DB, USERNAME, PASSWORD, {})
models = xmlrpc.client.ServerProxy(f'{URL}/xmlrpc/2/object')

# Ejemplo: actualizar una página por su ID
models.execute_kw(DB, uid, PASSWORD, 'website.page', 'write', [[60], {
    'website_meta_title': 'Implementación Odoo ERP | <tu-marca>',
    'website_meta_description': 'Descripción orientada a buscadores, de 120 a 155 caracteres.',
    'website_meta_keywords': 'odoo, erp, implementación, <tu-mercado>'
}])
print("Metadatos actualizados.")
```

### 3. Definir un robots.txt limpio

```python
# update_robots_txt.py
import xmlrpc.client

URL = 'https://<tu-dominio-prod>'
DB = '<tu-db>'
USERNAME = '<tu-usuario>'
PASSWORD = '<tu-password>'

CLEAN_ROBOTS = """User-agent: *
Allow: /
Disallow: /web/login
Disallow: /web/signup
Disallow: /web/reset_password
Disallow: /my/
Disallow: /web/database/

Sitemap: https://<tu-dominio-prod>/sitemap.xml
"""

common = xmlrpc.client.ServerProxy(f'{URL}/xmlrpc/2/common')
uid = common.authenticate(DB, USERNAME, PASSWORD, {})
models = xmlrpc.client.ServerProxy(f'{URL}/xmlrpc/2/object')

website_ids = models.execute_kw(DB, uid, PASSWORD, 'website', 'search', [[]])
models.execute_kw(DB, uid, PASSWORD, 'website', 'write',
                  [website_ids, {'robots_txt': CLEAN_ROBOTS}])
print("robots.txt actualizado.")
```

## Desindexado del Staging con Nginx

Un error común es dejar el entorno de pruebas indexable por Google. Si el *staging* compite con producción por las mismas keywords, estás canibalizando tu propio tráfico.

La solución fue marcar el vhost de staging con `noindex` de forma agresiva, a nivel de cabeceras HTTP:

```nginx
add_header X-Robots-Tag "noindex, nofollow, noarchive" always;
```

Y serví un `robots.txt` con `Disallow: /` para no dejar dudas. Además, bloqueé el `sitemap.xml` del staging (HTTP 404) para que Google no encuentre un mapa de sitio con URLs duplicadas:

```nginx
location = /sitemap.xml {
    return 404;
}
```

Con esto, el *staging* desaparece del índice mientras que la producción conserva todo el peso.

## llms.txt: Indexación para Agentes de IA

Los buscadores tradicionales ya no son los únicos clientes de tu web. Los agentes de IA y los motores de búsqueda conversacionales leen contenido estructurado. El estándar **`llms.txt`** es un archivo de texto plano, legible para máquinas, que describe qué contiene tu sitio.

También lo sirvo desde Nginx, con CORS abierto para que cualquier agente pueda consumirlo:

```nginx
location = /llms.txt {
    root /var/www/mi-sitio;
    default_type text/markdown;
    add_header Content-Type "text/markdown; charset=utf-8";
    add_header Access-Control-Allow-Origin "*" always;
    expires 1d;
}
```

## Lecciones Aprendidas

1. **El sitemap es tuyo, no del framework.** Si tu página sobrevive al índice de Google gracias a un archivo, ese archivo debe estar bajo tu control, no generado dinámicamente por un módulo de demo.

2. **Los metadatos se automatizan o se olvidan.** Con 17 páginas, un script de auditoría marca al instante qué página quedó sin título o descripción. El UI manual no escala.

3. **Staging y producción no comparten el índice.** Un `X-Robots-Tag: noindex` en el staging evita dolores de cabeza de duplicados y de peso diluido.

4. **Un ERP puede ser un CMS potente.** Odoo Community, correctamente vitaminizado con la capa de presentación adecuada (Nginx, landing pages, jerarquía semántica), compite sin las restricciones de licencia de la versión Enterprise.

5. **La accesibilidad y el SEO van de la mano.** Un solo H1 y una jerarquía clara no es solo una regla de SEO: también es mejor para tus lectores y para los agentes que leen tu HTML.

## Referencias

* **Odoo Website SEO.** Documentación oficial de optimización de motores de búsqueda en Odoo. [https://www.odoo.com/documentation/latest/applications/websites/website/maintenance.html](https://www.odoo.com/documentation/latest/applications/websites/website/maintenance.html)

* **Nginx ngx_http_headers_module.** Documentación de cabeceras HTTP (`add_header`, `expires`). [https://nginx.org/en/docs/http/ngx_http_headers_module.html](https://nginx.org/en/docs/http/ngx_http_headers_module.html)

* **ESTÁNDAR llms.txt.** Propuesta abierta para mejorar el contenido dirigido a LLM. [https://llmstxt.org/](https://llmstxt.org/)

* **Odoo External API (XML-RPC).** Documentación oficial de la API externa. [https://www.odoo.com/documentation/latest/developer/reference/external_api.html](https://www.odoo.com/documentation/latest/developer/reference/external_api.html)

* **Google Search Central — robots.txt.** Guía oficial para archivos robots.txt y las directivas `noindex`. [https://developers.google.com/search/docs/crawling-indexing/robots/intro](https://developers.google.com/search/docs/crawling-indexing/robots/intro)
