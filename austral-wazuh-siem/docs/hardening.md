# Hardening: respuesta, detección de vulnerabilidades, integridad y retención
**AustralPay es una organización ficticia de laboratorio.**

Resumen de las capacidades de respuesta y cumplimiento. Detalle paso a paso en [`bitacora/bloque-4-respuesta-hardening.md`](./bitacora/bloque-4-respuesta-hardening.md).

## 1. Active Response — bloqueo automático de atacantes

- Respuesta automatizada que **bloquea la IP de origen** de un brute force SSH (firewall-drop) tras dispararse la detección.
- **Graduada por confianza:** la acción se ata a alertas de severidad suficiente, evitando bloqueos por ruido.
- **Whitelist de hosts confiables:** los orígenes internos legítimos (incluida `austral-ws01`) están exentos, para no auto-bloquear operación válida.
- Evidencia de ejecución: [`../evidence/eventos/active-responses-4A.log`](../evidence/eventos/active-responses-4A.log).

> **Nota de diseño (y hallazgo del capstone):** la misma whitelist que protege de auto-bloqueos es el punto ciego que un atacante *dentro* de un host confiable explota (ver INC-2026-001, movimiento lateral con credencial válida). La confianza por IP de origen corta para los dos lados.

## 2. Detección de vulnerabilidades

- Módulo de Vulnerability Detection activo: inventaría los paquetes de cada agente y los cruza contra feeds de CVE.
- Da visibilidad del riesgo de exposición de los hosts (superficie de ataque conocida).

## 3. FIM — monitoreo de integridad de archivos (whodata)

- FIM con **whodata** sobre directorios sensibles (p. ej. `/opt/austral-app/config`, y en el capstone `/opt/austral-app/data`).
- Whodata no solo detecta el cambio: registra **quién** (usuario, proceso) lo hizo, vía el subsistema de auditoría.
- En el capstone, este FIM produjo la evidencia forense clave: la cadena `soporte-pagos → sudo → tee → root` reconstruida en un solo evento (alerta 550).

## 4. Retención de logs — PCI-DSS Req. 10

- Rotación y retención de logs configurada (logrotate) alineada al **Requisito 10 de PCI-DSS** (registro y monitoreo de accesos), coherente con el contexto de una fintech que maneja datos de pago.
- Justifica el archivado de eventos de auditoría y define su ciclo de vida.

## Cierre

Estas capacidades convierten al SIEM de un sistema que *solo detecta* en uno que **detecta, responde y cumple** — y que registra con trazabilidad de usuario, condición necesaria para responder a un incidente real con datos regulados.
