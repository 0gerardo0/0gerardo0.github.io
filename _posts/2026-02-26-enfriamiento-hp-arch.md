---
title: "Ingeniería Térmica en Laptops HP: Gestión de Recursos y Control de ACPI en Arch Linux"
date: 2026-02-26 23:30:00 -0600
categories: [Sistemas, Hardware]
tags: [nbfc, arch-linux, performance, engineering, virtualization]
toc: true
image:
  path: /assets/img/posts/hp-thermal-optimization.png
  alt: "Resultados de frecuencia y temperatura bajo carga"
---

El hardware HP fabricado en 2015 presenta limitaciones físicas y de firmware severas bajo distribuciones Linux. El Controlador Embebido (EC) prioriza el silencio operativo sobre la integridad del procesador AMD A9-9420. El sistema ignora las señales térmicas estándar del kernel y provoca picos de 99°C, resultando en bloqueos del bus de datos y parálisis total del sistema.

## 1. Contención de Recursos y Memoria Virtual

La ejecución de aplicaciones basadas en Electron, como Antigravity, genera reservas masivas de memoria virtual. El uso inicial de `ulimit` para mitigar fugas de memoria resultó ineficaz, provocando errores de interrupción (*Trace/breakpoint trap*).

* **Implementación de cgroups v2**: Se sustituyó el límite de shell por contenedores de recursos mediante `systemd-run`.
* **Restricción Física**: Se limitó la memoria RAM real (RSS) a 4GB.
* **Control del Kernel**: El Out-Of-Memory (OOM) killer actúa sobre el proceso específico antes de que la saturación de memoria bloquee las interrupciones del hardware.

## 2. El Fracaso del Control Térmico Estándar

La primera fase de automatización utilizó un script básico que reiniciaba el servicio NBFC ante picos de calor. Los resultados demostraron que el firmware de HP recupera el control de los ventiladores en intervalos de 30 a 60 segundos, anulando cualquier configuración externa.

### Resultados de la Prueba Inicial (Falla)
| Configuración | Temperatura Media | Frecuencia Media |
| :--- | :--- | :--- |
| ACPI Stock | 83.1 °C | 3009 MHz |
| Guardian v1 | 90.6 °C | 2821 MHz |

El primer intento de control manual fue un 6.2% menos eficiente que el sistema de fábrica, debido a la latencia en la reasignación de registros del controlador.

## 3. NBFC Guardian Pro: Lógica de Insistencia

Para vencer la prioridad de la BIOS, se desarrolló el sistema **NBFC Guardian Pro**. Este motor implementa una técnica de fuerza bruta técnica: reinyecta el perfil de configuración hexadecimal directamente en los registros del hardware cada 30 segundos.

* **Histéresis Estructurada**: Se estableció un umbral de activación en 60°C y una zona de desactivación en 50°C. Esta brecha de 10 grados estabiliza los ciclos de corriente del motor del ventilador.
* **Anulación de ACPI**: Al concatenar los comandos `config` y `restart`, el script garantiza que el firmware reciba la orden de inicialización antes de que pueda reclamar la prioridad.

## 4. Evidencia Empírica y Benchmarks

Se realizaron pruebas de estrés sintético de 120 segundos comparando el firmware original contra el sistema Guardian Pro.

| Métrica | ACPI Stock | Guardian Pro | Mejora Neta |
| :--- | :--- | :--- | :--- |
| Frecuencia Sostenida | 1541 MHz | 2655 MHz | **+72.2%** |
| Frecuencia Mínima | 1.5 MHz | 1796 MHz | Estabilidad |
| Temperatura Máxima | 89.1 °C | 93.2 °C | Flujo Activo |

![Gráfica de Rendimiento](/assets/img/posts/hp-thermal-optimization.png)

Los datos confirman que el sistema ACPI original reduce la frecuencia del CPU a niveles críticos para mitigar el calor. El sistema Guardian Pro elimina el *thermal throttling* severo, manteniendo una potencia constante un 72% superior.

## 5. Arquitectura del Repositorio

El proyecto se organiza de forma modular para facilitar su despliegue en hardware similar:

* `scripts/nbfc-guardian.sh`: Lógica de monitoreo e inyección de perfiles.
* `scripts/nbfc-pro`: Interfaz de línea de comandos para gestión de estados.
* `scripts/antigravity-run`: Wrapper de ejecución bajo cgroups.
* `configs/HP_Preventive.json`: Mapeo de registros para el microcontrolador HP.

El sistema transforma un hardware restrictivo en una estación de trabajo funcional, extrayendo el rendimiento máximo teórico del procesador AMD A9 bajo entornos Arch Linux. El código completo y los datos crudos están disponibles en el repositorio [nbfc-guardian-pro](https://github.com/0gerardo0/nbfc-guardian-pro).
