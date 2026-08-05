# Bloque 2 — Agentes + Sysmon (Wazuh SIEM)
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Wazuh SIEM · AustralPay Lab
Sesiones: 2A, 2B

---

## Objetivo

Desplegar agentes Wazuh en los tres hosts monitoreados de la infraestructura (dos servidores Linux y un endpoint Windows), e integrar Sysmon en el endpoint Windows para telemetría de proceso enriquecida.

## Entorno

| Host | SO | IP | Rol |
|---|---|---|---|
| `austral-app` | Ubuntu Server 24.04.4 LTS | 192.168.1.55 | Servidor de aplicación de pagos |
| `austral-db` | Ubuntu Server 24.04.4 LTS | 192.168.1.60 | Base de datos |
| `austral-ws01` | Windows 10 Pro 22H2 (build 19045.3803) | 192.168.1.65 | Estación de operaciones |

| Componente | Detalle |
|---|---|
| Versión de agente Wazuh | v4.9.2 (fijada, alineada con el manager) |
| Registro de agentes | Automático, vía puerto 1515 |
| Telemetría adicional en Windows | Sysmon v15.21, configuración de SwiftOnSecurity |

## Qué se hizo

1. Despliegue de dos VMs Ubuntu Server 24.04 LTS (`austral-app`, `austral-db`) con IP estática y acceso SSH verificado.
2. Instalación del agente Wazuh en ambas, fijando explícitamente la versión `4.9.2` (alineada con el manager) y bloqueando actualizaciones automáticas del paquete.
3. Enrolamiento automático de ambos agentes contra el manager (`austral-siem`), verificado en estado **Active** desde el dashboard.
4. Despliegue de una VM Windows 10 Pro (`austral-ws01`) con IP estática, Escritorio remoto habilitado y actualizaciones de Windows en pausa.
5. Instalación del agente Wazuh para Windows (versión `4.9.2`), enrolado mediante la herramienta de auto-registro del propio agente.
6. Instalación de Sysmon con la configuración pública de referencia de SwiftOnSecurity.
7. Configuración del agente Wazuh para recolectar el canal de eventos de Sysmon (`Microsoft-Windows-Sysmon/Operational`).
8. Verificación en el dashboard de que los eventos de Sysmon llegan al manager y se correlacionan con técnicas del framework MITRE ATT&CK.

## Decisiones de diseño

- **Versión de agente fijada explícitamente (`4.9.2`)**: evita la desalineación entre agente y manager, que puede causar fallos de conectividad o comportamiento no soportado entre versiones mayores distintas.
- **Registro automático de agentes**: adecuado para un entorno de laboratorio controlado; en un despliegue con mayor superficie de exposición se optaría por registro manual o con credenciales de enrolamiento reforzadas.
- **Configuración de Sysmon de SwiftOnSecurity**: configuración de referencia ampliamente adoptada en la industria, que prioriza señales de alto valor para detección y reduce ruido, evitando reinventar reglas de filtrado desde cero.
- **Agente como transporte, no como motor de decisión**: el agente recolecta y envía telemetría; la lógica de detección (decoders y reglas) vive exclusivamente en el manager, lo que limita el impacto de un agente comprometido.

## Evidencia

- `evidence/02-agentes-conectados.png` — Los tres agentes (`austral-app`, `austral-db`, `austral-ws01`) en estado Active, versión de agente alineada con el manager.
- `evidence/03-sysmon-activo.png` — Vista de MITRE ATT&CK del dashboard filtrada por `austral-ws01`, mostrando alertas mapeadas a técnicas reales a partir de la telemetría de Sysmon.

![Agentes conectados](evidence/02-agentes-conectados.png)
![Sysmon activo con mapeo MITRE](evidence/03-sysmon-activo.png)

## Estado del checklist

- [x] Agentes desplegados en `austral-app` y `austral-db`
- [x] Agente y Sysmon desplegados en `austral-ws01`
- [x] Canal de eventos de Sysmon integrado a la recolección del agente
- [x] Eventos verificados llegando y correlacionados con MITRE ATT&CK

## Próximo paso

Bloque 3 — generación de telemetría de ataque (fuerza bruta SSH, fallos de RDP, PowerShell sospechosa), y desarrollo de reglas de detección personalizadas mapeadas a técnicas MITRE ATT&CK.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
