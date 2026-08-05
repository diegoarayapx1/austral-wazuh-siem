# Despliegue: SIEM + agentes + Sysmon
**AustralPay es una organización ficticia de laboratorio.**

Resumen del despliegue del mini-SOC. El detalle paso a paso (con evidencia y decisiones) está en la bitácora: [`bitacora/bloque-1-despliegue-siem.md`](./bitacora/bloque-1-despliegue-siem.md) y [`bitacora/bloque-2-agentes-sysmon.md`](./bitacora/bloque-2-agentes-sysmon.md).

## 1. Wazuh single-node (manager + indexer + dashboard)

- Desplegado con **Docker Compose** sobre Ubuntu Server 24.04, versión **4.9.2**.
- Los tres componentes (manager, indexer, dashboard) corren como contenedores en `austral-siem` (192.168.1.50).
- TLS entre componentes con los certificados generados por el tooling de Wazuh.
- Verificación: acceso al dashboard, indexer sano, manager recibiendo en 1514/1515.

## 2. Enrolamiento de agentes

Tres hosts monitoreados, enrolados contra el manager:

| Host | SO | Notas de enrolamiento |
|---|---|---|
| `austral-app` (.55) | Ubuntu | agente APT, versión fijada a 4.9.2 |
| `austral-db` (.60) | Ubuntu | agente APT, versión fijada a 4.9.2 |
| `austral-ws01` (.65) | Windows 10 | agente MSI + Sysmon |

**Lecciones clave documentadas:**
- **Version pinning:** el repo `stable` instala la última versión por defecto, desalineando el agente del manager. Solución: instalar versión explícita (`wazuh-agent=4.9.2-1`) + `apt-mark hold`.
- **Dirección del manager independiente de la key:** importar la key de enrolamiento con `agent-auth` NO configura la dirección del manager en `ossec.conf`; hay que fijar `<address>` manualmente o el agente falla con `ERROR (4112)`.

## 3. Sysmon en Windows

- Instalado en `austral-ws01` con la **configuración SwiftOnSecurity**.
- Aporta telemetría de creación de procesos (Event ID 1) con linaje padre-hijo, cambios de registro (EID 12/13), acceso a procesos (EID 10) y creación de archivos (EID 11).
- Wazuh recolecta el canal `Microsoft-Windows-Sysmon/Operational` y lo mapea a MITRE ATT&CK.
- **Lección (teclado español / VMware):** el backtick se corrompe a acento agudo al pegar en la consola VMware con layout español; se usan arreglos de strings con `Set-Content` en vez de here-strings.

## Resultado

Al cierre de esta fase: SIEM operativo, tres agentes reportando, y telemetría de endpoint enriquecida en Windows — la base sobre la que se construye la detección (bloque 3) y la respuesta (bloque 4).
