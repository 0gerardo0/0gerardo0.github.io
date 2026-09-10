---
title: "Validación de Sensores IoT con ZKP: Diseñando Circuitos en Circom y Setup con Groth16"
date: 2026-09-09 20:00:00 -0600
categories: [Criptografía, Proyectos]
tags: [zkp, snarks, groth16, circom, iot, arduino, python, seguridad]
toc: true
image:
  path: /assets/img/posts/zkp-sensor-cover.jpg
  alt: "Macro de circuito electrónico y microcontrolador — Fotografía por Alexandre Debiève en Unsplash"
---

> *Fotografía de portada: [Alexandre Debiève](https://unsplash.com/@alexandre_debieve) en [Unsplash](https://unsplash.com/photos/macro-photography-of-black-circuit-board-g10wW_e7y3Q) (Licencia Unsplash).*
{: .prompt-info}

¿Cómo demostrar que un sensor en un entorno industrial, ambiental o médico registró un valor dentro del rango operativo seguro **sin revelar la lectura exacta ni exponer el secreto industrial o los datos del paciente**? 

Esa es la pregunta central de mi proyecto de tesis: el diseño e implementación de un sistema de atestación de telemetría IoT respaldado por **Pruebas de Conocimiento Cero (Zero-Knowledge Proofs - ZKP)**. 

En esta primera entrega técnica, quiero compartir el roadmap de mi tesis, la arquitectura que diseñé, los gráficos comparativos frente a paradigmas convencionales y los avances recientes en la capa criptográfica: el diseño de mi primer circuito en **Circom 2.0** para comprobación de rangos numéricos, el análisis de restricciones algebraicas y la evaluación empírica con benchmarks estadísticos mediante el protocolo **Groth16**.

---

## El Problema: Integridad vs Privacidad en IoT

En los sistemas tradicionales de monitoreo, los dispositivos periféricos (microcontroladores como Arduino MEGA, ESP32 o Raspberry Pi) transmiten lecturas crudas a un servidor central o a la nube:

```
[ Sensor ] ───(Lectura en texto plano: 42.5°C)───► [ Servidor Central ]
```

Este enfoque presenta dos problemas críticos:
1. **Fuga de privacidad / confidencialidad:** Si el canal o la base de datos se ven comprometidos, cualquier observador conoce las lecturas exactas. En aplicaciones médicas, de infraestructura crítica o procesos industriales con propiedad intelectual, esos valores son datos sensibles.
2. **Vulnerabilidad a manipulación intermedia:** Aunque se implemente TLS/HTTPS, el servidor receptor debe procesar y confiar ciegamente en que el valor no fue alterado si las claves del dispositivo sufren una filtración o si existe un intermediario con acceso al payload.

### Comparativa de Paradigmas en Seguridad IoT

Para dimensionar las ventajas de este enfoque, comparé el modelo ZKP frente a los dos estándares industriales dominantes:

![Comparativa de Modelos de Seguridad e Integridad en IoT](/assets/img/posts/zkp-comparativa-paradigmas.png)
*Figura 1: Evaluación multidimensional de paradigmas de integridad y privacidad en IoT (Escala 1 a 5).*

* **Telemetría Cruda (HTTP/MQTT):** Es sumamente liviana para microcontroladores pequeños, pero carece por completo de privacidad e integridad demostrable ante terceros.
* **Canal Seguro Tradicional (TLS Centralizado):** Protege el tránsito punto a punto, pero no genera pruebas criptográficas verificables independientemente y expone el dato crudo al servidor central.
* **Validación ZKP (Groth16):** Otorga privacidad matemática total (cero revelación de la señal privada), garantiza integridad inmutable y desacopla la verificación en costo $O(1)$.

---

## Arquitectura del Sistema Propuesto

Para llevar esta teoría a la práctica sin asfixiar los recursos del microcontrolador (un microcontrolador de 8 bits como el ATmega2560 cuenta con solo 8 KB de SRAM y carece de FPU, haciendo inviable computar curvas elípticas pesadas en el propio chip), diseñé una arquitectura desacoplada en tres capas:

![Arquitectura Desacoplada: Sensor Físico a Prover Edge a Verifier](/assets/img/posts/zkp-arquitectura-desacoplada.png)
*Figura 2: Diagrama de flujo de datos y separación de responsabilidades entre el hardware sensorial y el entorno criptográfico.*

1. **Capa Sensorial (Arduino MEGA 2560 + DHT22):** Adquisición de señales físicas (temperatura, humedad). Escala las magnitudes con coma fija (ej. $24.5^\circ\text{C} \to 245$ unidades enteras) y emite la trama serial hacia el host perimetral.
2. **Capa Prover (Node.js / Circom / C++):** Toma la lectura escalada como señal privada (`val`) y los límites reglamentarios como señales públicas (`min`, `max`), computa el testigo (*witness*) y genera la prueba zk-SNARK mediante **Groth16**.
3. **Capa Verifier (Backend API):** Consume la prueba $\pi$, el vector de entradas públicas y la `verification_key.json` para dar el veredicto de validación en tiempo constante $O(1)$.

---

## Roadmap del Proyecto y Estado Actual

Actualicé el roadmap del repositorio ([zkp-sensor-validation-thesis](https://github.com/0gerardo0/zpk-sensor-validation)) para documentar los hitos alcanzados en esta etapa y el plan de trabajo:

```
[Fase 1: Fundamentos & Setup] ───► [Fase 2: Circuitos ZKP Base] ───► [Fase 3: Integración Hardware] ───► [Fase 4: Pipeline & Eval]
        [x] COMPLETADA                    [~] EN PROGRESO                      [ ] PENDIENTE                      [ ] PENDIENTE
```

| Fase | Descripción | Estado |
| :--- | :--- | :--- |
| **Fase 1: Fundamentos & Toolchain** | Configuración del entorno en Arch Linux, CLI de Arduino, Circom 2.1+, SnarkJS, scripts de diagnóstico y marco teórico. | **Completada** |
| **Fase 2: Circuitos ZKP & Setup** | Circuito PoC `hello-world`, circuito `range_check.circom`, Trusted Setup Powers of Tau y generación de llaves Groth16. | **Avanzado** |
| **Fase 3: Adquisición & Hardware** | Lectura física con DHT22 en Arduino MEGA, validación de ruido de sensor y comunicación serial con el Prover. | **Siguiente hito** |
| **Fase 4: Pipeline End-to-End** | Servicio de generación de pruebas en el edge y API de verificación en backend. | **Planificado** |
| **Fase 5: Métricas y Tesis** | Benchmarking de restricciones R1CS, consumo de memoria y redacción académica. | **Planificado** |

---

## Diseñando el Circuito: `range_check.circom`

El núcleo de la lógica en ZKP reside en expresar las reglas computacionales como un **Sistema de Restricciones de Rango 1 (R1CS)**. En lugar de escribir un condicional clásico tipo `if (val >= min && val <= max)`, se construyen relaciones cuadráticas de la forma $A \cdot B = C$ sobre un campo primo finito ($\mathbb{F}_p$).

Aprovechando la biblioteca `circomlib`, escribí el circuito `range_check.circom`:

```circom
pragma circom 2.0.0;

include "circomlib/circuits/comparators.circom";

template RangeCheck(n) {
    // Señal privada: la medición real del sensor
    signal input val;

    // Señales públicas: rango permitido de operación
    signal input min;
    signal input max;

    // Comparador: val >= min
    component geq = GreaterEqThan(n);
    geq.in[0] <== val;
    geq.in[1] <== min;

    // Comparador: val <= max
    component leq = LessEqThan(n);
    leq.in[0] <== val;
    leq.in[1] <== max;

    // Ambas condiciones deben evaluarse estrictamente a 1
    geq.out === 1;
    leq.out === 1;
}

// Instanciación del circuito para números de 32 bits, exponiendo min y max como públicos
component main {public [min, max]} = RangeCheck(32);
```

> **Detalle técnico clave:** En la línea final, declaro `{public [min, max]}`. Esto significa que quien verifique la prueba conocerá qué umbrales se exigieron (ej. `min = 18`, `max = 30`), pero la señal `val` se mantiene completamente oculta dentro de la prueba criptográfica.
{: .prompt-tip}

### Métricas de Complejidad del Circuito (R1CS)

Al compilar este circuito e inspeccionarlo con `snarkjs r1cs info`, obtuve las métricas exactas del sistema de restricciones sobre la curva `bn128`:

* **Wires (Cables/Variables):** 74
* **Constraints (Restricciones R1CS):** 74
* **Private Inputs:** 1 (`val`)
* **Public Inputs:** 2 (`min`, `max`)

Al tener únicamente 74 restricciones, el costo de generar la prueba y el tamaño de los parámetros es sumamente compacto, lo que facilita su cálculo en microcomputadores perimetrales (gateways industriales o SBCs).

---

## Compilación y Ceremonia de Setup Criptográfico

Para generar pruebas Groth16, se requiere un **Trusted Setup** (Ceremonia de Powers of Tau) que proporcione los parámetros criptográficos sobre la curva elíptica `bn128`.

### 1. Compilación a R1CS y Testigos
Compilé el circuito para generar las restricciones y el runtime en WebAssembly/C++ encargado de calcular los testigos:

```bash
circom src/zkp/circuits/range_check.circom --r1cs --wasm --sym -o src/zkp/circuits/
```

### 2. Powers of Tau & Generación de Llaves
Utilizando una ceremonia Powers of Tau de 12 bits (`pot12_final.ptau`), generé la llave del probador (`.zkey`) y exporté la llave de verificación pública:

```bash
# Generar la llave inicial del circuito (Groth16 setup)
snarkjs groth16 setup \
  src/zkp/circuits/range_check.r1cs \
  tools/ptau/pot12_final.ptau \
  src/zkp/circuits/range_check_0000.zkey

# Contribuir entropía a la fase 2 específica del circuito
snarkjs zkey contribute \
  src/zkp/circuits/range_check_0000.zkey \
  src/zkp/keys/range_check_final.zkey \
  --name="Gerardo ZKP Thesis Contributor" -v -e="random-entropy-seed"

# Exportar la Verification Key en formato JSON
snarkjs zkey export verificationkey \
  src/zkp/keys/range_check_final.zkey \
  src/zkp/keys/verification_key.json
```

---

## Evidencia Experimental y Rigor Criptográfico

Para garantizar que el circuito cumple con las propiedades formales de **completitud** (*completeness*) y **solidez computacional** (*computational soundness*), sometí el sistema a una batería de pruebas unitarias, análisis de casos frontera y pruebas de alteración (*tampering*):

### 1. Pruebas de Frontera y Completitud

Evalué el circuito ante casos límite en el dominio de números enteros sin signo:

* **Punto Interior Nominal:** $val = 24$, en $[18, 30] \implies$ **Genera prueba válida y Verifica OK.**
* **Frontera Inferior Exacta:** $val = 18$, $min = 18 \implies$ **Verifica OK.**
* **Frontera Superior Exacta:** $val = 30$, $max = 30 \implies$ **Verifica OK.**
* **Límite Cero:** $val = 0$, en $[0, 100] \implies$ **Verifica OK.**

### 2. Resiliencia ante Ataques de Falsificación (Soundness)

Probé intencionalmente generar testigos con mediciones fuera de rango:

* **Transgresión por 1 unidad abajo:** $val = 17$, $min = 18 \implies$ El motor de aserción falla de inmediato: `Error: Assert Failed (RangeCheck line 18)`.
* **Transgresión por 1 unidad arriba:** $val = 31$, $max = 30 \implies$ Falla inmediata: `Error: Assert Failed (RangeCheck line 19)`.
* **Ataque de Modificación de Señales Públicas (Tampering):** Si un atacante intercepta una prueba válida generada para $[18, 30]$ y altera el vector de entradas públicas a $[26, 30]$ para intentar validar retroactivamente una lectura restringida, `snarkjs.groth16.verify` **rechaza la prueba con veredicto `false`**.
* **Ataque de Mutación del Vector de Prueba:** Si un bit del punto $\pi_A$ en la curva elíptica es alterado, la ecuación de emparejamiento bilineal no cierra y la prueba es **rechazada (`false`)**.

### 3. Benchmarks Estadísticos de Rendimiento ($N = 50$)

Para evaluar la estabilidad temporal y latencia sin sesgos de arranque en frío (*cold-start*), ejecuté un muestreo continuo de $N = 50$ iteraciones del Prover y Verifier en Node.js sobre Linux (kernel x86_64):

![Distribución de Latencia y Estabilidad Temporal de Groth16](/assets/img/posts/zkp-benchmarks-latencia.png)
*Figura 3: Distribución de latencias (boxplot) y estabilidad temporal por corrida (N = 50).*

A continuación muestro los estadísticos descriptivos obtenidos tras las 50 corridas:

| Operación | Media ($\mu$) | Desv. Estándar ($\sigma$) | Mediana ($P_{50}$) | Percentil 95 ($P_{95}$) | Mínimo | Máximo |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Generación (Prover)** | $20.53\text{ ms}$ | $2.41\text{ ms}$ | $20.15\text{ ms}$ | $26.85\text{ ms}$ | $18.02\text{ ms}$ | $27.91\text{ ms}$ |
| **Verificación (Verifier)** | $6.71\text{ ms}$ | $0.86\text{ ms}$ | $6.58\text{ ms}$ | $7.62\text{ ms}$ | $5.90\text{ ms}$ | $11.12\text{ ms}$ |

> **Interpretación:** La verificación se mantiene extraordinariamente predecible con una mediana de **$6.58\text{ ms}$** y una dispersión submilimétrica ($\sigma = 0.86\text{ ms}$). Esto demuestra que un único núcleo de servidor o gateway puede auditar más de **140 atestaciones de sensores por segundo**.
{: .prompt-info}

---

## Consideraciones de Seguridad y Estado del Arte

Este trabajo está fundamentado en literatura primaria y herramientas estándar de la industria criptográfica:

1. **Groth, Jens (2016).** *"On the Size of Pairing-Based Non-Interactive Arguments"*. Publicado en EUROCRYPT 2016 y disponible en [IACR ePrint 2016/260](https://eprint.iacr.org/2016/260.pdf). Introduce el protocolo Groth16, demostrando que una prueba zk-SNARK puede reducirse a tan solo 3 elementos de grupo ($\pi_A \in G_1, \pi_B \in G_2, \pi_C \in G_1$) con verificación basada en emparejamientos bilineales (*pairings*).
2. **Buterin, Vitalik (2016).** *"Quadratic Arithmetic Programs: from Zero to Hero"*. Artículo fundamental que detalla la conversión paso a paso de computaciones aritméticas a R1CS y polinomios QAP.
3. **Iden3 (2023).** *Circom 2.0 Documentation & circomlib*. Documentación técnica del compilador de dominios específicos para circuitos de conocimiento cero ([docs.circom.io](https://docs.circom.io/)).
4. **RareSkills (2023).** *The RareSkills Zero Knowledge Book*. Guía técnica y práctica moderna sobre construcción y auditoría de circuitos en Circom.

### Supuestos Críticos y Trabajo Futuro

Como parte del rigor metodológico de mi tesis, tengo presentes las restricciones del prototipo en esta etapa:
* **Representación Numérica:** Las mediciones con decimales de sensores como el DHT22 deben representarse en punto fijo (ej. factor de escala $\times 10$ o $\times 100$) previo a ingresar al circuito, dado que $\mathbb{F}_p$ opera estrictamente con enteros modulares.
* **Atestación de Origen:** El circuito actual valida que *alguna lectura $val$* estuvo en rango, pero no firma la identidad del sensor. En la Fase 3 integraré una atestación física (hash encadenado o firma digital de la lectura) para mitigar el ataque de suplantación de identidad del sensor.

---

## Siguiente paso: Fase 3

Con los circuitos validados y las llaves verificadas, el siguiente paso es la **Fase 3: Integración de Hardware**. Voy a desarrollar el firmware del Arduino MEGA para muestrear el sensor DHT22, formatear las tramas en punto fijo y transmitirlas por puerto serial hacia el Prover.

Documentar este proyecto no solo me ayuda a estructurar los avances de mi tesis, sino a mostrar que la criptografía de conocimiento cero tiene aplicaciones prácticas y viables en el mundo físico y el IoT.

*El código fuente, los circuitos y el roadmap completo están disponibles en el repositorio [0gerardo0/zkp-sensor-validation-thesis](https://github.com/0gerardo0/zpk-sensor-validation).*
