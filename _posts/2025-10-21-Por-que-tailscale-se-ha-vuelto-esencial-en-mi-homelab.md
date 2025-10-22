---
title: "Adiós, CGNAT: Cómo Tailscale liberó mi red personal"
date: 2025-10-21 16:30:00 -0600
categories: [Redes, Self-Hosting]
tags: [tailscale, cgnat, redes, vpn, nextcloud, homelab]
toc: true
image:
  path: /assets/img/posts/tailscale-banner.png
  alt: "Logo de Tailscale conectando dispositivos en una red."
---

En el mundo del *self-hosting* y la administración de sistemas, hay un enemigo silencioso que puede frustrar tus planes antes de que empiecen: el **CGNAT**.

Durante meses, tuve la frustración de no poder acceder a los servicios de mi propia casa desde el exterior. Quería conectarme por SSH a mi servidor, revisar archivos o probar una aplicación web que estaba desarrollando, pero me topaba con una pared. Mi proveedor de internet, como muchos otros, me tiene detrás de una red CGNAT (Carrier-Grade NAT).

En este post, quiero hablar de este problema y de la herramienta que, para mí, ha sido la solución mágica: **Tailscale**.

## El Problema: ¿Qué es el CGNAT y por qué es una Pesadilla?

Imagina que tu casa es un gran edificio de apartamentos. Tu dirección IP pública es la dirección del edificio (ej. "Av. Palitos 666"). Tu router y tus dispositivos (laptop, teléfono) tienen direcciones internas (ej. "Apartamento 101", "Apartamento 102").

Normalmente, si quisieras acceder a tu servidor desde fuera, solo necesitarías saber la dirección del edificio y el número de tu apartamento (abriendo un puerto).

    Con CGNAT, la situación es técnicamente mucho más compleja y restrictiva.

CGNAT (Carrier-Grade NAT) es una solución a gran escala que usan los Proveedores de Servicios de Internet (ISP) para combatir el agotamiento de las direcciones IPv4. En lugar de asignarle a tu router una dirección IP pública única, el ISP te asigna una dirección IP privada de un rango reservado (comúnmente 100.64.0.0/10).

Esto crea un escenario de doble NAT:

    Tu NAT Local: Tu propio router traduce las IPs de tus dispositivos (ej. 192.168.1.50) a la IP privada que te dio el ISP (ej. 100.x.x.x).

    El NAT del ISP (CGNAT): Un router masivo del ISP toma el tráfico de cientos de clientes (todos en el rango 100.x.x.x) y lo traduce a una única dirección IP pública compartida.

**La Pesadilla:** El problema fundamental es que tú no tienes control sobre el router del ISP. Es técnicamente muy díficil configurar el reenvío de puertos (port forwarding) en ese dispositivo. Cualquier conexión entrante desde internet (como una solicitud SSH al puerto 22) llega a esa IP pública compartida, y el router del ISP no tiene forma de saber que ese tráfico está destinado a tu servidor específico.

Para cualquiera que quiera auto-alojar servicios, esto es un obstáculo total. Estás, en efecto, inaccesible e invisible desde el internet público.
## La Solución: ¿Qué es Tailscale y Cómo Funciona?

La belleza de Tailscale radica en su complejidad interna, pero su simplicidad externa. No necesitas entender criptografía ni redes; solo necesitas instalarlo.

### La Magia Interna: WireGuard y "Hole Punching"

A diferencia de las VPNs tradicionales que enrutan todo el tráfico a través de un servidor central, Tailscale crea una red de **malla (mesh network).**

1. **WireGuard:** En el corazón de Tailscale se encuentra WireGuard, un protocolo de VPN extremadamente rápido, moderno y seguro. Tailscale se encarga de gestionar automáticamente todas las claves de cifrado de WireGuard por ti.

2. **Servidores de Coordinación:** Cuando instalas Tailscale en un dispositivo (como tu servidor Arch Linux), este se registra con los servidores de coordinación de Tailscale usando tu cuenta (Google, GitHub, etc.). Estos servidores actúan como una "guía telefónica" segura.

3.  **"Hole Punching" (Perforación NAT):** Aquí es donde se vence al CGNAT.
    * Tu servidor en casa (detrás de CGNAT) mantiene una conexión saliente activa con el servidor de coordinación.
    * Tu teléfono (en la calle) hace lo mismo.
    * Cuando tu teléfono quiere conectarse a tu servidor, le pregunta al servidor de coordinación: "¿Dónde está el servidor de Arch?".
    * El servidor de coordinación, que conoce las rutas de ambos, les ayuda a "presentarse" y a establecer un **túnel cifrado directo P2P (Peer-to-Peer)**. El objetivo es que el tráfico fluya directamente entre tus dispositivos, lo que lo hace increíblemente rápido y privado.
    * En el raro caso de que las redes de ambos lados sean demasiado restrictivas e impidan una conexión P2P, Tailscale utiliza sus servidores de retransmisión cifrados (DERP) como respaldo, asegurando que la conexión siempre se establezca.

### Configuración: Mi Experiencia en Arch Linux y Android

Configurarlo es sorprendentemente simple y consistente en todas las plataformas.

#### En Arch Linux (Mi Servidor)

En Arch, la configuración es muy clara si distinguimos las dos partes que la componen: el servicio (`tailscaled`) y la herramienta de control (`tailscale`).

1.  **Instalación:**
    Primero, instalamos el paquete desde los repositorios oficiales.
    ```bash
    sudo pacman -S tailscale
    ```

2.  **Iniciar y Habilitar el Servicio:**
    Este es el paso crítico. Necesitamos que el servicio (daemon) `tailscaled` se inicie automáticamente con el sistema para que nuestro servidor siempre esté en la red después de un reinicio.
    ```bash
    sudo systemctl enable --now tailscaled
    ```
3.  **Conectar a la Red:**
    Ahora que el servicio está corriendo, usamos la herramienta de cliente `tailscale` para conectar la máquina a nuestra red:
    ```bash
    sudo tailscale up
    ```
    Este comando imprimirá una URL en la terminal. Solo tienes que copiar esa URL, pegarla en tu navegador, iniciar sesión con tu cuenta (ej. GitHub), y el servidor aparecerá mágicamente en tu red.

#### En Android (Mi Teléfono)
1.  **Instalación:** Ve a la Google Play Store y descarga la aplicación oficial de Tailscale.
2.  **Autenticación:** Abre la aplicación. Te pedirá iniciar sesión con la **misma cuenta** que usaste en tu servidor (ej. GitHub).
3.  **Conectar:** Activa el interruptor principal dentro de la aplicación.

¡Eso es todo! En menos de 5 minutos, ambos dispositivos pueden verse mutuamente usando sus direcciones IP `100.x.x.x` asignadas por Tailscale, sin importar en qué parte del mundo se encuentren.

#### En cualquier otro servicio (Docker, VMs, etc.)
El proceso es idéntico. Ya sea que tengas un contenedor de Docker (para Nextcloud), una máquina virtual o un Raspberry Pi, solo tienes que instalar el cliente de Tailscale correspondiente y ejecutar `tailscale up`. El dispositivo se unirá a tu red privada y será accesible al instante.

## ¿Cómo me ayuda a mí? (El Beneficio Real)

Gracias a Tailscale, mi red casera ya no está aislada. Ahora puedo:
* **Conectarme por SSH** a mi servidor desde cualquier lugar del mundo simplemente usando su IP de Tailscale (`ssh gerardo@100.x.x.x`).
* **Acceder a servicios web** que estoy desarrollando en mi máquina local (como si estuviera en `localhost`).
* **Transferir archivos** de forma segura entre mi teléfono y mi servidor sin pasar por una nube de terceros.

Resolvió mi problema fundamental de accesibilidad de una forma increíblemente simple y segura.

## Mi Propósito a Futuro: El Servidor de Nextcloud

Todo esto era una preparación para mi verdadero objetivo: montar mi propio servidor de **Nextcloud**.

Nextcloud es una plataforma de código abierto para auto-alojar tus propios archivos, calendarios, contactos y fotos (básicamente, tu propia nube privada).

El problema era: ¿de qué sirve una nube privada si no puedo acceder a ella desde mi teléfono cuando estoy fuera de casa?

Con Tailscale, el problema está resuelto. Instalaré Nextcloud en mi servidor y solo será accesible a través de su IP de Tailscale. Esto me da lo mejor de ambos mundos:
1.  **Accesibilidad Global:** Puedo sincronizar mis archivos y fotos desde mi teléfono en cualquier lugar, siempre que tenga Tailscale activado.
2.  **Seguridad Total:** Mi servidor de Nextcloud no estará expuesto en absoluto a la internet pública, evitando ataques de bots y hackers.

Para cualquiera que se haya sentido frustrado por el CGNAT, recomiendo de corazón probar Tailscale. Ha sido un cambio total en las reglas del juego para mis proyectos de *self-hosting*.

## Referencias

* Pennarun, A. (2020, 20 marzo). ***How Tailscale works.*** Tailsacale blog. [https://tailscale.com/blog/how-tailscale-works](https://tailscale.com/blog/how-tailscale-works)

* GeeksforGeeks. (2025, 12 julio). ***NAT Hole Punching in Computer Network.*** GeeksforGeeks. [https://www.geeksforgeeks.org/computer-networks/nat-hole-punching-in-computer-network/](https://www.geeksforgeeks.org/computer-networks/nat-hole-punching-in-computer-network/)

* GetPublicIP Ltd. (s. f.). ***Self hosting behind carrier grade NAT (CGNAT).*** [https://getpublicip.com/guides/self-hosting/self-hosting-behind-carrier-grade-nat-cgnat](https://getpublicip.com/guides/self-hosting/self-hosting-behind-carrier-grade-nat-cgnat)

* Nextcloud. (s. f.). ***Nextcloud - Open source content collaboration platform.*** [https://nextcloud.com/](https://nextcloud.com/)

* ***CGNAT Killed Your Static IP—Get Back Online with a 10-Minute Cloudflare Tunnel*** – GeoSaffer.com. (2025, 3 julio). [https://blog.geosaffer.com/2025/07/03/203/](https://blog.geosaffer.com/2025/07/03/203/)

* Donenfeld, J. A. (s. f.). ***WireGuard: fast, modern, secure VPN tunnel.*** [https://www.wireguard.com/#simple-network-interface](https://www.wireguard.com/#simple-network-interface)
