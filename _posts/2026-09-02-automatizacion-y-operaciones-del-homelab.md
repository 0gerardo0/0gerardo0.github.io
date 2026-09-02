---
title: "Automatización y Operaciones en mi Homelab: Backups, Monitoreo y Despliegue Continuo"
date: 2026-09-02 06:00:00 -0600
categories: [Infraestructura, DevOps]
tags: [homelab, docker, ansible, backups, monitoreo, linux, debian]
toc: true
layout: post
image:
  path: /assets/img/posts/homelab-ops-banner.png
  alt: "Diagrama de arquitectura del homelab"
---

Un homelab sin automatización es solo un servidor encendido esperando a que algo falle. Después de meses de operar Odoo, Nextcloud y varios servicios en un solo servidor Debian, llegué a una conclusión: los scripts sueltos y los manuales de comandos no escalan. Lo que necesitaba era un sistema coherente que cubriera tres pilares: **respaldo**, **observabilidad** y **despliegue**.

Este post documenta cómo construí esa infraestructura de operaciones, los errores que cometí en el camino y las herramientas que finalmente funcionaron.

## El Problema: Operar a Mano

Mi servidor original era una colección de comandos sueltos en la terminal. Para hacer un backup de Nextcloud, ejecutaba un `rsync` manual. Para verificar que Odoo estuviera arriba, hacía un `curl` y rezaba. Para desplegar un cambio en el tema, copiaba archivos con `scp`.

El primer indicio de que esto no funcionaba fue cuando perdí una configuración de Nginx editada directamente en el servidor. Sin control de versiones, sin backup previo, sin forma de saber qué había cambiado. Ese día entendí que la operación manual es un riesgo, no una alternativa.

## La Arquitectura de Operaciones

El sistema que construí se apoya en tres capas que se complementan:

| Capa | Herramienta | Función |
| :--- | :--- | :--- |
| **Respaldo** | Scripts Bash + cron | Copias automáticas de Odoo y Nextcloud |
| **Observabilidad** | curl + Telegram Bot | Alertas de salud y métricas de respuesta |
| **Despliegue** | Git + Ansible + SSH | Flujo controlado de cambios al servidor |

## Capa 1: Backups Automatizados

La primera regla de un homelab serio: **si no está respaldado, no existe**. Implementé dos scripts separados para Odoo y Nextcloud, cada uno con sus particularidades.

### Backups de Odoo

Odoo almacena datos en dos lugares: la base de datos PostgreSQL y el filestore en `/var/lib/odoo/`. Un backup incompleto es un backup inútil.

```bash
#!/bin/bash
# backup-odoo.sh — Backup completo de Odoo
set -euo pipefail

BACKUP_DIR="/var/backups/odoo"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="odoo"

# Backup de la base de datos
pg_dump "$DB_NAME" | gzip > "$BACKUP_DIR/db_${TIMESTAMP}.sql.gz"

# Backup del filestore
tar -czf "$BACKUP_DIR/filestore_${TIMESTAMP}.tar.gz" \
    -C /var/lib/odoo filestore/

# Retención: eliminar backups de más de 7 días
find "$BACKUP_DIR" -name "*.gz" -mtime +7 -delete

echo "[$(date)] Backup completado: $TIMESTAMP"
```

El punto clave es la **retención automática**. Sin ella, los backups acumulados llenan el disco en semanas. Un ciclo de 7 días es suficiente para la mayoría de homelabs personales.

### Backups de Nextcloud

Nextcloud es más complejo. Necesitas respaldar la base de datos, los archivos y la configuración. Además, el contenedor Docker debe estar en modo de mantenimiento durante el backup para evitar inconsistencias.

```bash
#!/bin/bash
# backup-nextcloud.sh — Backup completo de Nextcloud en Docker
set -euo pipefail

BACKUP_DIR="/home/gera/nextcloud/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Activar modo mantenimiento
docker exec nextcloud php occ maintenance:mode --on

# Backup de la base de datos
docker exec nextcloud-db mysqldump -u root \
    --password="secreto" nextcloud > "$BACKUP_DIR/db_${TIMESTAMP}.sql"

# Backup de archivos y configuración
tar -czf "$BACKUP_DIR/nextcloud_${TIMESTAMP}.tar.gz" \
    -C /mnt/storage/nextcloud config/ data/

# Backup del docker-compose
cp /home/gera/nextcloud/nextcloud-docker-compose.yml \
    "$BACKUP_DIR/compose_${TIMESTAMP}.yml"

# Desactivar modo mantenimiento
docker exec nextcloud php occ maintenance:mode --off

# Retención
find "$BACKUP_DIR" -name "*.gz" -mtime +7 -delete
find "$BACKUP_DIR" -name "*.sql" -mtime +7 -delete

echo "[$(date)] Nextcloud backup completado: $TIMESTAMP"
```

> **Nota:** El modo de mantenimiento es obligatorio. Si omites esta operación, los archivos pueden cambiar durante la copia, generando una base de datos inconsistente con el filestore.
{: .prompt-info }

Ambos scripts se ejecutan via `cron` a las 3:00 AM, cuando el uso del servidor es mínimo.

```bash
# /etc/crontab
0 3 * * * root /home/gera/scripts/backup-odoo.sh >> /var/log/backup-odoo.log 2>&1
5 3 * * * root /home/gera/scripts/backup-nextcloud.sh >> /var/log/backup-nextcloud.log 2>&1
```

## Capa 2: Observabilidad y Alertas

Los backups te protegen contra la pérdida de datos. Pero ¿cómo sabes si tu servidor está funcionando correctamente en tiempo real? Mi solución fue un sistema de monitoreo ligero basado en scripts que verifican el estado de los servicios y envían alertas a Telegram.

### Health Check Centralizado

En lugar de instalar un agente de monitoreo completo (Prometheus, Grafana), construí un script que prueba los endpoints críticos y reporta el estado:

```bash
#!/bin/bash
# health-check.sh — Verificación de servicios
set -euo pipefail

TELEGRAM_TOKEN="tu_token"
CHAT_ID="tu_chat_id"

check_service() {
    local name="$1" url="$2"
    local status
    status=$(curl -s -o /dev/null -w "%{http_code}" \
        --max-time 10 "$url" 2>/dev/null || echo "000")
    
    if [ "$status" -ge 200 ] && [ "$status" -lt 400 ]; then
        echo "✅ $name: OK ($status)"
    else
        echo "❌ $name: FAIL ($status)"
        send_alert "$name no responde (HTTP $status)"
    fi
}

send_alert() {
    local message="$1"
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage" \
        -d chat_id="$CHAT_ID" \
        -d text="⚠️ Homelab Alert: $message" \
        -d parse_mode="Markdown" > /dev/null
}

check_service "Odoo" "https://0gerardo0.engineer/web/login"
check_service "Nextcloud" "https://cloud.0gerardo0.engineer/login"
check_service "Nginx" "http://localhost/health"
```

### Métricas de Rendimiento

Además del estado, necesitaba saber qué tan rápido responden los servicios. El TTFB (Time To First Byte) es una métrica simple pero poderosa:

```bash
# Medición de TTFB para Nextcloud
curl -o /dev/null -w "TTFB: %{time_starttransfer}s\n" \
    https://cloud.0gerardo0.engineer/index.php/204
```

Un TTFB superior a 2 segundos indica un problema. En mi caso, el primer diagnóstico reveló que Nextcloud estaba generando logs excesivos que saturaban el disco. La solución fue configurar `logrotate` y reducir el nivel de logs de la aplicación.

## Capa 3: Despliegue Continuo

Esta es la capa que más transformó mi flujo de trabajo. El principio es simple: **nunca editar archivos directamente en el servidor**. Todo cambio pasa por Git.

### El Flujo

```
Local → git commit → git push → Servidor: git pull → Reiniciar servicio
```

Para implementar esto, configuré un repositorio en el servidor con los archivos de configuración de Nginx, Docker Compose y los scripts de operaciones. Cada cambio se commitea localmente, se pushea, y en el servidor se ejecuta un `git pull` seguido de la reinicialización del servicio afectado.

### Ansible para Configuración Reproducible

Para los cambios más complejos, como la instalación inicial o la configuración de seguridad, utilicé Ansible. Un playbook de Ansible documenta exactamente qué hace cada paso, es reproducible y funciona como documentación viva.

```yaml
# playbook de ejemplo: configuración de Nginx
- name: Configurar Nginx para Odoo
  hosts: servidor
  become: yes
  tasks:
    - name: Copiar configuración de Nginx
      template:
        src: templates/odoo-nginx.conf.j2
        dest: /etc/nginx/sites-available/0gerardo0
      notify: Reload Nginx

    - name: Habilitar sitio
      file:
        src: /etc/nginx/sites-available/0gerardo0
        dest: /etc/nginx/sites-enabled/0gerardo0
        state: link

  handlers:
    - name: Reload Nginx
      service:
        name: nginx
        state: reloaded
```

> **Nota:** Ansible no es necesario para un homelab pequeño. Pero si tienes más de dos servicios, la inversión en aprendizaje se paga rápido. La alternativa es mantener un README con comandos, que inevitablemente se desactualiza.
{: .prompt-info }

## Lecciones Aprendidas

Después de operar este sistema durante varios meses, estas son las lecciones más importantes:

1. **Los backups sin verificación no son backups.** Ejecuta un restore de prueba al menos una vez al mes. Un backup que no puedes restaurar es un archivo que ocupa espacio.

2. **La simplicidad supera a la robustez.** Un script de Bash con `curl` y un bot de Telegram puede ser más confiable que un stack completo de monitoreo, especialmente en un servidor con recursos limitados.

3. **Git como fuente de verdad.** Si un archivo de configuración no está en Git, no existe. Esto aplica para Nginx, Docker Compose, scripts de backup y cualquier otra cosa que puedas necesitar replicar.

4. **Los logs son tu primera línea de diagnóstico.** Configurar `logrotate` desde el día uno te ahorra problemas de disco lleno. Un servidor con 8GB de RAM y 50GB de disco no tolera logs descontrolados.

5. **Automatiza el aburrimiento, no la creatividad.** Los backups, health checks y despliegues deben ser automáticos. La configuración de nuevos servicios y la experimentación deben seguir siendo manuales, al menos hasta que entiendas qué estás haciendo.

## Conclusiones

Operar un homelab no es solo tener un servidor encendido. Es construir un sistema que se mantenga solo, que te alerte cuando algo falle y que te permita dormir tranquilo sabiendo que tus datos están respaldados.

El camino desde scripts sueltos hasta un sistema coherente de operaciones no fue lineal. Hubo backups corruptos, alertas falsas y despliegues que rompieron servicios. Pero cada error enseñó algo nuevo, y el resultado final es una infraestructura que puedo reconstruir desde cero si es necesario.

La clave no está en las herramientas específicas, sino en la disciplina: todo en Git, todo automatizado, todo verificado.

> **Actualización:** este post se publicó originalmente con el blog en Chirpy v7.3.1; tras migrar a v7.6.0, todos los enlaces y la configuración de despliegue siguen funcionando sin cambios.

## Referencias

* **Ansible.** Documentación oficial de automatización de infraestructura. [https://docs.ansible.com/](https://docs.ansible.com/)

* **Docker Compose.** Referencia para definir y ejecutar aplicaciones Docker multi-contenedor. [https://docs.docker.com/compose/](https://docs.docker.com/compose/)

* **Nginx.** Documentación de configuración de proxy reverso y seguridad. [https://nginx.org/en/docs/](https://nginx.org/en/docs/)

* **PostgreSQL.** Guía de backup y recuperación. [https://www.postgresql.org/docs/current/backup-dump.html](https://www.postgresql.org/docs/current/backup-dump.html)

* **Cron.** Manual de programación de tareas en Linux. [https://man7.org/linux/man-pages/man8/cron.8.html](https://man7.org/linux/man-pages/man8/cron.8.html)

* **Telegram Bot API.** Documentación para envío de mensajes programáticos. [https://core.telegram.org/bots/api](https://core.telegram.org/bots/api)
