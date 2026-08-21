# SIEM Wazuh: Aprendiendo monitoreo de seguridad en mi homelab

- **Fecha:** 2026-08-21
- **Entorno:** Nodo Proxmox con un contenedor LXC (Debian 13, 4 vCPU / 6 GB RAM / 50 GB disco), Wazuh 4.14.7 all-in-one (indexer + manager + dashboard), red overlay Tailscale.
- **Estado:** Funcionando. Dos agentes enrolados (Windows 11 y macOS en Apple Silicon), reportando de forma remota; detección de vulnerabilidades activa.

## Objetivo

Aprender qué hace realmente un SIEM desplegando uno en mi propio hardware y apuntándolo a mis propias máquinas. No un demo de curso: un stack real y funcionando que vigila mis notebooks y me enseña cómo se comportan en la práctica las alertas, los logs y los feeds de vulnerabilidades.

## Por qué un SIEM (y por qué Wazuh)

Un SIEM centraliza los logs de muchas máquinas y los convierte en preguntas: quién falló al iniciar sesión, qué cambió en disco, qué software está desactualizado y expuesto. Wazuh es una buena opción open source para aprender porque el stack completo (indexer + manager + dashboard) corre en hardware modesto y el agente soporta Windows, macOS y Linux.

La arquitectura en una línea: los agentes recolectan, el manager analiza con reglas, el indexer almacena, el dashboard visualiza.

## Qué hice

1. **Desplegué Wazuh en un contenedor LXC de mi nodo Proxmox.** La instalación all-in-one es un solo script instalador, con un detalle real: el indexer OpenSearch necesita `vm.max_map_count`, un parámetro de kernel que en un contenedor no privilegiado se debe configurar en el host, no dentro del invitado.
2. **Enrolé mis dos máquinas de uso diario como agentes:** un notebook con Windows 11 (paquete MSI) y un notebook macOS en Apple Silicon (pkg, más un permiso manual de Full Disk Access para que el agente pueda leer los logs unificados).
3. **Hice los agentes alcanzables desde cualquier lado con Tailscale.** El manager se une a la red overlay y ambos agentes reportan a través de ella, así el monitoreo sigue funcionando cuando las máquinas están fuera de mi LAN.
4. **Activé la detección de vulnerabilidades** en la config del manager (bloque `<vulnerability-detection>`) y dejé que descargara los feeds CVE y corriera un primer escaneo.

## Qué observé

- Ambos agentes reportan como **Activos** y sus eventos aparecen en el dashboard en tiempo real: inicios de sesión (incluidos los fallidos, que el motor de reglas marca tras intentos repetidos), lanzamientos de procesos, cambios de archivos.
- El **escaneo de vulnerabilidades** fue lo más instructivo. El primer escaneo tras la actualización de feeds CVE dejó clara la foto de mi higiene de parches:
  - el agente macOS reportó un set grande de CVEs a nivel de SO, la mayoría ya resueltos en versiones más nuevas de macOS;
  - el agente Windows marcó un puñado de CVEs a nivel de apps en software instalado antiguo.
- Nada exótico: sin zero-days, solo la historia de siempre de "tus sistemas están atrasados en actualizaciones" que cualquier SIEM te va a contar. Que es exactamente el punto de tener uno.

## Lecciones

- **Los feeds CVE SON el detector.** La detección de vulnerabilidades no sirve hasta que se descargan los feeds; el escaneo recién tomó sentido tras una actualización de feeds. "Actualizar los feeds" es lo primero que hay que revisar cuando un escaneo se ve vacío.
- **Los hallazgos no son alertas.** Los resultados de vulnerabilidades van a un índice dedicado y a la pestaña Vulnerability Detection del dashboard; no generan eventos de alerta necesariamente. "Sin alertas" no significa "no está escaneando".
- **Un SIEM es un espejo de tus hábitos de actualización.** La forma más rápida de bajar los números es parchear, y el SIEM lo verifica después. También es un buen argumento para tener backups y un calendario de actualizaciones.
- **Los parámetros de kernel pertenecen al host.** Cualquier servicio que necesite mucha memoria mapeada en un LXC (OpenSearch, Elasticsearch...) necesita `vm.max_map_count` configurado en el host Proxmox.
- **El dimensionamiento importa.** Las specs "mínimas" oficiales asumen decenas de agentes; un homelab con dos notebooks corre el stack completo cómodo con 4 vCPU / 6 GB RAM.

## Próximos pasos

- Ajustar reglas para que los hallazgos importantes generen alertas y no vivan solo en el índice.
- Agregar más fuentes de logs (mi router, el propio host Proxmox).
- Explorar la API para armar scripts simples sobre los datos.

## Verificación

```bash
# en el manager (LXC Debian):
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard   # todos activos
/var/ossec/bin/agent_control -l                                   # agentes en Active
# dashboard: pestaña Agents -> ambas máquinas Active
#            pestaña Vulnerability Detection -> inventario CVE por agente
```
