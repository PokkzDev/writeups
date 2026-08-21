# Liberación del CPE de Mundo Pacífico: Desactivación de TR-069 + Bridge manual

- **Fecha:** 2026-08-05
- **Fuente:** Luis, procedimiento realizado en su propio equipo (línea residencial, Chillán, fibra GPON). Verificado en vivo.
- **Estado:** Funcionando. ONT en bridge, TR-069/CWMP desactivado, el router propio gestiona WAN/firewall/DHCP.

## Objetivo

Tomar control total del router/ONT provisto por el ISP (Mundo Pacífico, línea con CGNAT):

1. Impedir que el ISP envíe configuraciones de forma remota (canal TR-069 / CWMP).
2. Poner el equipo en modo bridge para que mi propio router se encargue del firewall, DHCP, DNS y Wi-Fi, sin señales en conflicto y sin doble NAT.
3. Mantener la línea funcionando durante el cambio: sin llamadas a soporte, sin reprovisionamiento.

## Contexto

Los ISP gestionan su CPE mediante TR-069 (CWMP). El router se conecta **hacia afuera** al ACS del ISP por el **TCP 7547** y el ACS envía configuraciones: firmware, SSID, VLAN, aprovisionamiento. El router viene con credenciales de fábrica que se reemplazan en el primer aprovisionamiento.

La ventana entre el estado de fábrica y el primer contacto con el ACS es el momento más útil del equipo: una condición de carrera contra el sistema de aprovisionamiento. El equipo queda sin gestionar, sin aprovisionar y con credenciales por defecto hasta que el ACS gana la carrera.

## Procedimiento

1. **Reset de fábrica del router OFFLINE** (sin fibra/internet conectado). Arranca con valores de fábrica, credenciales de administrador por defecto y nada aprovisionado aún.
2. **Iniciar sesión con la contraseña de fábrica.**
3. **Reconectar la línea y capturar el primer push de configuración** cuando el ACS comienza a aprovisionar.
4. **Desactivar el canal TR-069/CWMP**: eliminar/deshabilitar la URL del ACS y los ajustes de inform para que no se puedan enviar más configuraciones ni firmware.
5. **Bridge del ONT**: configurar la WAN en modo bridge para que mi router termine la WAN directamente.
6. **Verificar**: el router obtiene la IP WAN, firewall/DHCP/DNS en mi equipo, el ONT queda como un bridge mudo.

## Resultados

- **WAN**: directa en mi router, dirección CGNAT como se esperaba (el tráfico entrante sigue vía Tailscale; es la columna vertebral de todos modos).
- **Red**: un solo punto de control. Sin doble NAT, sin DNS del ISP en el camino, sin superposición de Wi-Fi entre equipos.
- **Gestión del ISP**: canal remoto muerto. Sin más envíos de configuración, sin resets remotos, sin actualizaciones de firmware sorpresa.

## Advertencias / Lo que sigue aplicando

- **OMCI sigue existiendo**: en GPON el OLT gestiona el ONT a nivel de la capa PON vía OMCI; ese camino es parte del handshake de la fibra y no se puede desactivar desde la interfaz web. Matar TR-069 elimina ~95% de la gestión remota, no el 100%.
- **El firmware queda congelado**: sin TR-069 no hay actualizaciones automáticas. Aceptable porque el equipo ahora es un bridge mudo detrás de mi propio firewall.
- **Las llamadas a soporte pueden revertirlo**: una visita de técnico o un reprovisionamiento puede resetear el ONT. Después no hay ACS que lo re-aprovisione, así que la reconstrucción sería manual.
- **Respaldar la configuración**: archivo de backup del admin + detalles de la WAN (VLAN, credenciales PPPoE si el plan las usa). Sin TR-069, un segundo reset de fábrica significa reconstrucción manual.
- **Vigilar tras reinicios**: confirmar que TR-069 siga desactivado; algunos firmware lo reactivan desde configuración volátil. Señal: la URL del ACS de vuelta en la página CWMP.

## Verificación (chequeo rápido)

Comparar la IP WAN del router con la IP externa (ifconfig.me). Diferentes → CGNAT (RFC 6598 100.64.0.0/10), Tailscale sigue siendo el camino de entrada. Idénticas → IP pública real, el tráfico entrante se vuelve posible.

## Lecciones

- La ventana de aprovisionamiento (fábrica → primer contacto con el ACS) es una superficie de ataque real. Correr contra ella es una técnica legítima de liberación de CPE.
- La "gestión" del ISP es un límite de confianza que vale la pena auditar en tu propio equipo.
