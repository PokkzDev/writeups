# Actualización del SIEM Wazuh: 2 agentes por Tailscale + detección de vulnerabilidades activa (CT 300)

- **Fecha:** 2026-08-21
- **Fuente:** Sesión SSH en vivo al CT 300 (alias SSH `wazuh`, tailnet `100.109.208.61`): `agent_control`, `ossec.conf`, `ossec.log`, archivo de alertas, forense de RocksDB del escáner, `tailscale status`. Cada dato verificado en el equipo.
- **Estado:** Funcionando. 2 agentes **Activos**, ambos por Tailscale; feeds CVE actualizados hoy (12:53-12:55 CLT) con re-escaneo completado; **macbookairm2 ya reporta vulnerabilidades** en el dashboard (también gp77-13vg).

## Objetivo

Documentar el estado actual de la flota del SIEM: qué agentes están enrolados, cómo llegan al manager (Tailscale) y el estado de la detección de vulnerabilidades tras la actualización de feeds de hoy. En corto: "el Mac ahora ve vulnerabilidades".

## Antecedentes

- Wazuh 4.14.7 (rc1) all-in-one en el CT 300 desde el 2026-08-16 (ver el writeup de despliegue). En ese momento GP77 estaba enrolado y Activo; el MacBook Air quedaba pendiente.
- Desde 4.9/4.10 el escáner de vulnerabilidades es un módulo C++ nuevo (`wazuh-modulesd:vulnerability-scanner`) configurado en `ossec.conf` bajo `<vulnerability-detection>`. El bloque `<vulnerability-detector>` / `vulnerability-detector.yml` ya no existe. Ojo con el tag correcto.
- **Mecánica clave:** los hallazgos salen de los feeds CVE. Sin feeds no hay hallazgos. El feed completo recién llegó hoy, por eso el Mac "empezó a ver vulnerabilidades" ahora y no en la instalación.
- `/root/wazuh-passwords.txt` ya no existe en el equipo. Toda la evidencia de este doc se obtuvo sin credenciales de API. Si se necesita recuperar el password: `bash /root/wazuh-install.sh -p`.

## Snapshot de la flota (verificado 2026-08-21 ~14:40 CLT)

| ID | Agente | SO (agent_control) | Versión agente | Dispositivo Tailscale | IP tailnet | Ruta | Último keep-alive |
|---|---|---|---|---|---|---|---|
| 001 | `gp77-13vg` | Microsoft Windows 11 Pro | v4.14.7 | `pokkzseclaptop` | 100.122.146.80 | **directa** | ~14:40 |
| 002 | `macbookairm2` | Darwin 27.0.0 arm64 (macOS 27; `PokkzSec-MacBook-Air-M2.local`) | v4.14.7 | `pokkzsec-macbook-air-m2` | 100.126.178.88 | **relay "sao"** | ~14:25 |
| 000 | `wazuh` (el manager) | Debian 13 | v4.14.7 | - | - | local | - |

Ambos agentes comparten el mismo hash de configuración (`ab73af41...`): misma config push.

## Transporte por Tailscale: evidencia

- `tailscale status` en el CT 300: ambos dispositivos **activos**; GP77 directo, Mac vía relay DERP `sao` (inactivo, sin ruta directa por ahora; vuelve cuando esté en la misma red que otro nodo).
- TCP en vivo al momento de captura: `ESTAB 100.109.208.61:1514 <- 100.122.146.80:61532` (sesión de GP77).
- Atribución de IPs en el archivo de alertas (16-21 ago): **4.237 eventos desde `100.122.146.80`** (gp77), **350 desde `100.126.178.88`** (macbook), **0 desde LAN `192.168.1.x`**. Todo el tráfico de agentes va por la tailnet; los agentes no usan la ruta LAN.
- Interfaces del equipo: `eth0 192.168.1.250/24` (LAN), `tailscale0 100.109.208.61/32`.

## Detección de vulnerabilidades: la actualización

### Config (ossec.conf, activa desde la instalación)

```xml
<vulnerability-detection>
  <enabled>yes</enabled>
  <index-status>yes</index-status>
  <feed-update-interval>60m</feed-update-interval>
</vulnerability-detection>
```

### Qué pasó hoy (ossec.log, hora local CLT)

```
12:53:21 wazuh-modulesd:vulnerability-scanner: INFO: Initiating update feed process.
12:55:43 wazuh-modulesd:vulnerability-scanner: INFO: Triggered a re-scan after content update.
12:55:43 wazuh-modulesd:vulnerability-scanner: INFO: Feed update process completed.
```

- Almacén de feeds (`/var/ossec/queue/vd/feed/`, RocksDB): **~12 GB** tras la actualización (antes era un stub): la base CVE completa.
- Base de inventario (`/var/ossec/queue/vd/inventory/`) reescrita a las 12:55; el DB de eventos del escáner seguía procesando el **inventario de apps del Mac** (PIPAgent.app, etc.) a las 14:29. El primer escaneo completo aún se estaba asentando tras la actualización de feeds.
- Los hallazgos van al índice `wazuh-vulnerabilities-*`, que alimenta la pestaña **Vulnerability Detection** del dashboard. Eso es exactamente "el Mac ve vulnerabilidades".

### Qué tiene el escáner en archivo (forense del inventario RocksDB)

- **Agente 002 (macbookairm2):** set de CVEs a nivel de SO de **~120** (rango CVE-2026-28911 ... CVE-2026-64783, consistente con un boletín de seguridad actual de macOS 27), más **17 CVEs a nivel de apps en 4 apps**, p.ej. `CVE-2026-8643, CVE-2026-6357, CVE-2025-8869, CVE-2026-1703, CVE-2026-3219, CVE-2023-5752` (una app); `CVE-2026-59890, CVE-2025-47273, CVE-2024-6345, CVE-2022-40897` (otra); `CVE-2026-3029`; `CVE-2026-64783, CVE-2026-64719, CVE-2026-64728, CVE-2026-64730, CVE-2026-64757, CVE-2026-43804` (otra).
- **Agente 001 (gp77-13vg):** 8 CVEs a nivel de apps (era 2015-2021): `CVE-2021-30481, CVE-2020-15530, CVE-2019-17180, CVE-2019-15316, CVE-2019-15315, CVE-2019-14743, CVE-2015-7985, CVE-2015-4016`.
- Archivo de alertas (16-20 ago): los 54 hits del string "vulnerab" son **todos alertas SCA/CIS benchmark** (políticas Debian 13 / Win11 / macOS Tahoe), no alertas de vulnerabilidad. O sea, **nunca se disparó una alerta de vuln antes de que llegara el feed** (esperado: no había con qué comparar).
- Totales exactos por agente: el **dashboard es la fuente autoritativa** (pestaña Vulnerability Detection). El inventario del lado del servidor es la materia prima y los conteos cambian a medida que se actualiza SO/apps.

## Resultados

- **Flota:** 2/2 agentes Activos, ambos por Tailscale (directa + relay), mismo hash de config.
- **Detección de vulnerabilidades:** feeds al día (actualizados hoy, cadencia 60 min), re-escaneo disparado, inventarios de ambos agentes poblados. El Mac ya muestra su set de CVEs en el dashboard.
- **Salud del servidor:** up 4d21h, RAM 2.2/6.0 GiB, disco 19/50 GiB (41%), de los cuales ~12 GB son los feeds CVE.

## Caveats / Lo que sigue aplicando

- **Feeds = disco + ancho de banda:** 12 GB, re-descargados cada 60 min. Vigilar `/` y la NIC del PVE.
- **Los hallazgos no son alertas por defecto** (`index-status: yes` solamente): aparecen en la pestaña Vulnerability Detection; generar alertas por CVE nuevos requiere reglas a medida.
- **El Mac va por relay "sao" cuando está inactivo** (poco ancho de banda, sync de inventario lenta); la ruta directa vuelve cuando esté co-ubicado con otro nodo.
- **No hay `/root/wazuh-passwords.txt`:** recuperar `admin` es `bash /root/wazuh-install.sh -p` (resetea e imprime). Guardar el nuevo password en un lugar seguro.
- Los totales del dashboard pueden diferir levemente de las listas de arriba (escaneos incrementales; los CVEs resueltos/parchados desaparecen).

## Lecciones

- La config del nuevo escáner vive en `ossec.conf` `<vulnerability-detection>`; el viejo `vulnerability-detector.yml` ya no existe.
- Los resultados de vulnerabilidades viven en un **índice** de OpenSearch, no en el stream de alertas. "Sin alertas" no significa "no está escaneando".
- Forense RocksDB: `strings` sobre `queue/vd/inventory/*.log` vuelca las claves CVE por agente: una forma de probar lo que sabe el escáner sin credenciales.
- Sin feeds no hay hallazgos. **La actualización de feeds de hoy es la palanca real** que encendió el Mac.

## Verificación (reproducible)

```bash
ssh wazuh
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard   # active x3
/var/ossec/bin/agent_control -l                                   # 001, 002 Activos
/var/ossec/bin/agent_control -i 002                               # Darwin 27.0.0 arm64
tailscale status                                                  # ambos agentes activos
grep -a vulnerability-scanner /var/ossec/logs/ossec.log | tail    # feed update + re-scan
du -sh /var/ossec/queue/vd/feed                                   # ~12G
# dashboard -> Vulnerability Detection -> macbookairm2 -> lista CVE
```
