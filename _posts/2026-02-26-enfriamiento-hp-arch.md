---
title: "Domando una laptop HP (2015): Optimización Térmica Extrema en Arch Linux"
date: 2026-02-26 23:30:00 -0600
categories: [Linux, Hardware]
tags: [nbfc, arch-linux, performance, engineering]
toc: true
image:
  path: /assets/img/posts/hp-thermal-optimization.png
  alt: "Gráfica comparativa de rendimiento térmico"
---

Tu laptop HP no es lenta. Está asfixiada por su propio software. Este proyecto demuestra cómo puedes recuperar el control del hardware cuando el fabricante te lo impide.

## El origen del caos
El sistema sufría bloqueos totales. No podías mover el ratón ni usar el teclado. Los logs revelaron temperaturas de 99°C y errores de firmware APIC ID. Tu procesador AMD A9 bajaba su velocidad a 1.5 MHz para no quemarse. El sistema se congelaba porque el hardware se rendía ante el calor.

## La batalla contra la memoria
Intentamos limitar aplicaciones pesadas con ulimit. Electron y Chromium fallaron con errores de interrupción (Trace/breakpoint trap). Estas aplicaciones reservan terabytes de memoria virtual aunque no los usen.

Usamos systemd-run para solucionar esto.
* Creamos un contenedor de recursos con cgroups v2.
* Limitamos la RAM física a 4GB.
* El kernel mata la aplicación si excede el límite. Tu PC ya no se bloquea.

## Ingeniería contra el ACPI
El firmware de HP prioriza el silencio sobre la vida de tu procesador. El ACPI de fábrica apaga los ventiladores cada minuto aunque el CPU esté ardiendo.

Creamos NBFC Guardian Pro. Es un servicio de usuario que vigila tu hardware cada 5 segundos. Nuestro script es más terco que tu BIOS.
* Reinyecta tu perfil de configuración cada 30 segundos.
* Usa una brecha de 10 grados (60°C activación / 50°C reposo).
* El ventilador ya no tartamudea.

## Datos reales del campo de batalla
Hicimos pruebas de estrés reales. Comparamos el sistema de fábrica contra NBFC Guardian Pro.

| Métrica | Sistema ACPI (Stock) | NBFC Guardian Pro | Mejora |
| :--- | :--- | :--- | :--- |
| Frecuencia Media | 1541 MHz | 2655 MHz | **+72% de potencia** |
| Frecuencia Mínima | 1.5 MHz | 1796 MHz | **Estabilidad Total** |
| Temperatura Máxima | 89.1 °C | 93.2 °C | **Control Forzado** |

El sistema ACPI deja caer la frecuencia a niveles absurdos. Con Guardian Pro mantienes 2655 MHz constantes. Lograste un 72% más de potencia real bajo carga máxima.

## Estructura del sistema
El repositorio contiene las herramientas necesarias para replicar esto.
* **nbfc-guardian.sh:** El cerebro que vigila la temperatura.
* **nbfc-pro:** Tu herramienta de control en la terminal.
* **antigravity-run:** El protector de memoria para tus aplicaciones.

Tu hardware viejo ahora funciona como una estación de trabajo moderna. Tienes el control total de los ventiladores. Tu memoria RAM ya no colapsará el kernel.

Puedes encontrar el código completo aquí: [nbfc-guardian-pro](https://github.com/0gerardo0/nbfc-guardian-pro)
