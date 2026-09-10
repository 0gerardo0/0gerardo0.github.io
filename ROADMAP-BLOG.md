# Roadmap Blog + AdSense

Estado al: 2026-09-05

## Hecho

- [x] Página de Política de Privacidad (`/privacy-policy/`)
- [x] Aviso de consentimiento de cookies (GDPR/CCPA)
- [x] `ads.txt` con el Publisher ID de AdSense
- [x] Meta tag `google-adsense-account` en el `<head>`
- [x] Script de AdSense activado en `_config.yml` (`enabled: true` + client `ca-pub-8716410790967635`)
- [x] Banners de los posts reemplazados por imágenes de stock (seo-nginx-odoo, homelab-ops)

## Pendiente

- [ ] **Conseguir el `data-ad-slot`** de la unidad de anuncio en AdSense (Anuncios → Unidades de anuncios → display responsivo). Con eso se activa el bloque de anuncio dentro de los posts (`_includes/ad-banner.html`). Hoy solo corren los anuncios automáticos.
- [ ] **Mover el blog al dominio raíz `0gerardo0.engineer`** para que AdSense apruebe la revisión. Hoy la raíz la ocupa el staging de Odoo (con `noindex`), por eso AdSense dice "No se encuentra".
  - [ ] Decidir si el staging de Odoo se mueve a `staging.0gerardo0.engineer` o se queda como está (decisión de negocio, no tocar sin confirmar).
  - [ ] Cambiar DNS en Cloudflare: `0gerardo0.engineer` → GitHub Pages (`0gerardo0.github.io`).
  - [ ] Cambiar custom domain en GitHub Pages: `blog.0gerardo0.engineer` → `0gerardo0.engineer`.
  - [ ] Actualizar `CNAME` y `url` en `_config.yml` (hoy `https://blog.0gerardo0.engineer`).
  - [ ] Redirigir `blog.0gerardo0.engineer` → raíz (301) para no perder enlaces viejos.
- [ ] **Reenviar el dominio a revisión de AdSense** una vez que la raíz sirva el blog.
- [ ] **Verificar aprobación** y que los anuncios se rendericen (curl + página real).

## Notas

- Los posts del blog ya documentan el SEO técnico: `2026-09-05-seo-tecnico-con-nginx-y-odoo.md`.
- El staging de Odoo está desindexado a propósito (`X-Robots-Tag: noindex`), por eso no sirve para AdSense desde la raíz.