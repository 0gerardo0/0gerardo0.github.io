---
title: "Ingeniería Térmica en Laptops HP: Gestión de Recursos y Control de ACPI en Arch Linux"
date: 2026-02-26 23:30:00 -0600
categories: [Sistemas, Hardware]
tags: [nbfc, arch-linux, performance, engineering, virtualization]
toc: true
image:
  path: /assets/img/posts/acpi-v-nbfc-2.png
  alt: "Resultados de frecuencia y temperatura bajo carga"
---

El hardware HP fabricado en 2015 presenta limitaciones físicas y de firmware severas bajo distribuciones Linux. El Controlador Embebido (EC) prioriza el silencio operativo sobre la integridad del procesador AMD A9-9420. El sistema ignora las señales térmicas estándar del kernel y provoca picos de 99°C, resultando en bloqueos del bus de datos y parálisis total del sistema.

## El Desafío del Firmware y Hardware

El análisis de logs del sistema (`journalctl`) reveló inconsistencias críticas en la comunicación entre el kernel y el hardware:
* **Bugs de Firmware**: Errores persistentes de `APIC ID mismatch` y fallos en la resolución de símbolos `ACPI` (`_SB.WLBU._STA.WLVD`).
* **Inestabilidad del Bus**: Las fugas de recursos en aplicaciones pesadas provocaban interrupciones en el bus USB (`usb1-port2: Cannot enable`), inhabilitando periféricos de entrada antes del colapso total.
* **Prioridad de Silencio**: El EC sobrescribe cualquier instrucción de software cada 30-60 segundos para mantener el ventilador en RPM mínimas, incluso bajo estrés térmico.

## Contención de Recursos y Memoria Virtual

La ejecución de aplicaciones basadas en Electron, como Antigravity, genera reservas masivas de memoria virtual. El uso inicial de `ulimit` para mitigar fugas de memoria resultó ineficaz, provocando errores de interrupción (*Trace/breakpoint trap*).

* **Implementación de cgroups v2**: Se sustituyó el límite de shell por contenedores de recursos mediante `systemd-run`.
* **Restricción Física**: Se limitó la memoria RAM real (RSS) a 4GB, permitiendo el mapeo virtual extendido sin colapsar el sistema.
* **Control del Kernel**: El Out-Of-Memory (OOM) killer actúa sobre el proceso específico antes de que la saturación de memoria bloquee las interrupciones del hardware.

## El Fracaso del Control Térmico Estándar

La primera fase de automatización utilizó un script básico que reiniciaba el servicio NBFC ante picos de calor. Los resultados demostraron que el firmware de HP recupera el control de los ventiladores en intervalos cortos, anulando cualquier configuración externa.

### Resultados de la Prueba Inicial (Falla de Guardian v1)
| Configuración | Temperatura Media | Frecuencia Media |
| :--- | :--- | :--- |
| ACPI Stock | 83.1 °C | 3009 MHz |
| Guardian v1 | 90.6 °C | 2821 MHz |

El primer intento de control manual fue un 6.2% menos eficiente que el sistema de fábrica, debido a la latencia en la reasignación de registros del controlador.

## NBFC Guardian Pro: Lógica de Insistencia

Para vencer la prioridad de la BIOS, se desarrolló el sistema **NBFC Guardian Pro**. Este motor implementa una técnica de fuerza bruta técnica: reinyecta el perfil de configuración hexadecimal directamente en los registros del hardware cada 30 segundos.

* **Histéresis Estructurada**: Se estableció un umbral de activación en 60°C y una zona de desactivación en 50°C. Esta brecha de 10 grados estabiliza los ciclos de corriente del motor del ventilador.
* **Anulación de ACPI**: Al concatenar los comandos `config` y `restart`, el script garantiza que el firmware reciba la orden de inicialización antes de que pueda reclamar la prioridad.

## Evidencia Empírica y Benchmarks

Se realizaron pruebas de estrés sintético de 120 segundos (`stress --cpu 4`) monitoreadas con `s-tui`.

### Resultados de la Prueba Inicial (Falla de Guardian v1)

| Configuración | Temperatura Media | Frecuencia Media |
| :------------ | :---------------- | :--------------- |
| ACPI Stock    | 83.1 °C           | 3009 MHz         |
| Guardian v1   | 90.6 °C           | 2821 MHz         |

![Gráfica de Rendimiento V1](/assets/img/posts/acpi-v-nbfc-1.png)
_Figura 1: Comparativa de estabilidad de frecuencia y gestión de temperatura en su primera versión._

El primer intento de control manual fue un 6.2% menos eficiente que el sistema de fábrica. Esto se debió a la latencia en la reasignación de registros del controlador.

## Resultados Finales bajo Estrés

| Métrica              | ACPI Stock  | Guardian Pro | Mejora Neta      |
| :------------------- | :---------- | :----------- | :--------------- |
| Frecuencia Sostenida | 1541.4 MHz  | 2655.2 MHz   | **+72.2%**       |
| Frecuencia Mínima    | 1.5 MHz     | 1796.7 MHz   | Estabilidad      |
| Temperatura Máxima   | 89.1 °C     | 93.2 °C      | Flujo Activo     |

![Gráfica de Rendimiento V2](/assets/img/posts/acpi-v-nbfc-2.png)
_Figura 2: Comparativa de estabilidad de frecuencia y gestión de temperatura en su segunda versión._

Los datos confirman que el sistema ACPI original reduce la frecuencia del CPU a niveles críticos (Throttling) para mitigar el calor. El sistema Guardian Pro mantiene una potencia constante un 72% superior, extrayendo el rendimiento máximo teórico del procesador AMD A9.

## Lecciones de Ingeniería

1. **Persistencia sobre Configuración**: En hardware con ACPI restrictivo, no basta con configurar; es necesario automatizar la re-inyección constante de estados.
2. **Cgroups v2**: Es la herramienta definitiva para contener aplicaciones Electron/Chromium en Linux, superando las limitaciones de `ulimit`.
3. **Hardware Limitado**: Un equipo de 2015 no es necesariamente obsoleto; a menudo está limitado por una gestión térmica conservadora diseñada para entornos de oficina silenciosos.

El código fuente, los scripts de automatización y los datos crudos de las pruebas están disponibles en el repositorio [nbfc-guardian-pro](https://github.com/0gerardo0/nbfc-guardian-pro).

## Referencias y Documentación

La implementación se basa en protocolos de comunicación de bajo nivel y herramientas de monitoreo estándar.

**Estándares de Hardware y Energía**
* **Especificación ACPI (Advanced Configuration and Power Interface).** Protocolos de comunicación entre OS y firmware. [UEFI Forum](https://uefi.org/specifications).
* **Controlador Embebido (EC).** Guía técnica sobre microcontroladores en laptops. [Kernel.org](https://www.kernel.org/doc/html/latest/admin-guide/acpi/index.html).

**Herramientas de Control y Monitoreo**
* **NBFC-Linux (NoteBook FanControl).** Control de ventiladores mediante registros EC. [GitHub Repository](https://github.com/nbfc-linux/nbfc-linux).
* **s-tui (Stress-Terminal UI).** Interfaz de monitoreo para frecuencia y temperatura. [GitHub Repository](https://github.com/amanusk/s-tui).
* **Stress.** Generador de carga sintética para pruebas de estabilidad. [Debian Manpages](https://manpages.debian.org/testing/stress/stress.1.en.html).

**Gestión de Recursos en Linux**
* **Systemd Resource Control.** Gestión de límites de memoria mediante cgroups. [Freedesktop Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html).
* **Control Groups (cgroups v2).** Aislamiento y limitación de recursos del kernel. [Kernel.org Guide](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html).

**Especificaciones del Procesador**
* **AMD A9-9420.** Detalles sobre límites térmicos y frecuencias nominales. [AMD Product Specs](https://www.amd.com/en/support/downloads/drivers.html/processors/a-series/a9-series-apu-for-laptops/7th-gen-a9-9420-apu.html).
