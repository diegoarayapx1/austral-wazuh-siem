# AustralPay — Mini-SOC con Wazuh (Proyecto 1)

> **AustralPay es una organización FICTICIA creada exclusivamente con fines de laboratorio y portafolio. Nada en este repositorio corresponde a una empresa, infraestructura o incidente real. Todo el trabajo — despliegue, ataque, detección, respuesta y documentación — es propio, realizado de forma autónoma con fines de aprendizaje.**

---

## Qué es esto

Un **mini-SOC funcional** montado desde cero sobre **Wazuh SIEM**, para una fintech ficticia de laboratorio (AustralPay). El proyecto recorre el ciclo completo de un centro de operaciones de seguridad: desplegar la plataforma, dar visibilidad a los equipos, generar ataques reales contra el laboratorio, escribir detección propia, automatizar respuesta, y finalmente **ejecutar una intrusión completa e investigarla como un analista N1**.

El diferenciador del proyecto no es solo que funcione, sino el **criterio**: distinguir señal de ruido, reconstruir un ataque a partir de pocas alertas, y ser honesto sobre lo que la detección no ve.

📄 Resumen ejecutivo (poco tecnicismo): [`Brief-Ejecutivo-MiniSOC-AustralPay.pdf`](./Brief-Ejecutivo-MiniSOC-AustralPay.pdf)

---

## Entorno de laboratorio

| Host | IP | Rol | SO / Telemetría |
|---|---|---|---|
| `austral-siem` | 192.168.1.50 | Wazuh manager (Docker single-node, v4.9.2) | Ubuntu Server 24.04 |
| `austral-app` | 192.168.1.55 | Servidor de aplicación de pagos (objetivo) | Ubuntu + agente |
| `austral-db` | 192.168.1.60 | Servidor DB / máquina atacante | Ubuntu + agente |
| `austral-ws01` | 192.168.1.65 | Estación de operaciones (punto de entrada) | Windows 10 + Sysmon (SwiftOnSecurity) |
| Kali | 192.168.1.100 | Máquina atacante (origen externo) | Herramientas ofensivas de lab |

---

## El recorrido (5 bloques)

| # | Bloque | Qué se construyó | Detalle |
|---|---|---|---|
| 1 | Fundamentos + despliegue | Wazuh single-node en Docker, reproducible | [`docs/bitacora/bloque-1-despliegue-siem.md`](./docs/bitacora/bloque-1-despliegue-siem.md) |
| 2 | Agentes + Sysmon | 3 equipos reportando; telemetría enriquecida en Windows | [`docs/bitacora/bloque-2-agentes-sysmon.md`](./docs/bitacora/bloque-2-agentes-sysmon.md) |
| 3 | Telemetría de ataque + reglas | Ataques reales + reglas de detección custom | [`docs/bitacora/bloque-3a-telemetria-ataque.md`](./docs/bitacora/bloque-3a-telemetria-ataque.md) · [`docs/mitre-mapping.md`](./docs/mitre-mapping.md) |
| 4 | Respuesta + hardening | Active Response, Vuln Detection, FIM, retención PCI-DSS | [`docs/bitacora/bloque-4-respuesta-hardening.md`](./docs/bitacora/bloque-4-respuesta-hardening.md) · [`docs/hardening.md`](./docs/hardening.md) |
| 5 | Cadena de ataque + respuesta | Intrusión de 8 tácticas, informe N1, capa Navigator | ver **Capstone** abajo |

---

## Capstone — INC-2026-001

La pieza central: una **intrusión multi-etapa completa** (phishing → ejecución → persistencia → robo de credencial → reconocimiento → movimiento lateral → escalada a root → impacto sobre datos de pago), ejecutada contra el laboratorio y reconstruida como un **informe de incidente estilo analista N1**.

- 📋 Informe de incidente: [`incidents/INC-2026-001-cadena-intrusion.md`](./incidents/INC-2026-001-cadena-intrusion.md)
- 🖼️ Anexo visual de evidencias (E01–E12): [`incidents/Anexo-Evidencias-INC-2026-001.pdf`](./incidents/Anexo-Evidencias-INC-2026-001.pdf)
- ⚔️ Diseño y ejecución de la cadena: [`attack-chain/escenario-ataque.md`](./attack-chain/escenario-ataque.md)
- 🗺️ Capa MITRE ATT&CK Navigator: [`attack-chain/navigator-layer.json`](./attack-chain/navigator-layer.json)

**Cobertura de detección (honesta):** de 13 técnicas en 10 tácticas, **4 detectadas**, 3 contexto/parcial, **6 gaps** documentados con su mitigación. El hallazgo central: una alerta de nivel 12 pasó sin investigar 2.5 h antes del impacto (*missed detection opportunity*), y atenderla habría detenido la cadena.

---

## Reglas de detección propias

| Regla | Nivel | Detecta | MITRE |
|---|---|---|---|
| 100210 | 12 | PowerShell download-cradle (IEX + descarga) | T1059.001 |
| 100211 | 0 | Tuning FP (`__PSScriptPolicyTest_`) | — |
| 100212 | 0 | Tuning FP (`cleanmgr` / `WimProvider.dll`) | — |
| 100220 | 12 | Brute force SSH exitoso (fallo→éxito misma IP) | T1110.001 / T1078 |
| 100230 | 12 | Cuenta válida desde IP no confiable | T1078 |

Archivo: [`rules/local_rules.xml`](./rules/local_rules.xml) · Mapeo completo: [`docs/mitre-mapping.md`](./docs/mitre-mapping.md)

---

## Estructura del repositorio

```
austral-wazuh-siem/
├── README.md                     # este archivo
├── Brief-Ejecutivo-...pdf        # resumen ejecutivo (venta)
├── docs/
│   ├── arquitectura.md           # topología y componentes del lab
│   ├── despliegue.md             # despliegue del SIEM + agentes + Sysmon
│   ├── hardening.md              # Active Response, Vuln, FIM, retención PCI-DSS
│   ├── mitre-mapping.md          # reglas y cadena mapeadas a MITRE ATT&CK
│   └── bitacora/                 # bitácora por bloque (detalle del proceso)
├── rules/
│   ├── local_rules.xml           # reglas de detección custom
│   └── malicious-ips.list        # CDB list (placeholder, se puebla en Proyecto 4)
├── incidents/
│   ├── INC-2026-001-...md         # informe de incidente N1
│   └── Anexo-Evidencias-...pdf    # anexo visual de evidencias
├── attack-chain/
│   ├── escenario-ataque.md        # diseño + log de ejecución de la cadena
│   ├── timeline.md                # timeline de la intrusión
│   └── navigator-layer.json       # capa ATT&CK Navigator
└── evidence/                      # capturas
    ├── E01–E12 ...png             # capstone INC-2026-001
    ├── despliegue/                # agentes conectados, dashboard (bloques 1-2)
    ├── deteccion/                 # ataques, detección, reglas custom (bloque 3)
    ├── respuesta/                 # active response, vuln, FIM (bloque 4)
    └── eventos/                   # telemetría de soporte (JSON + logs)
```

> La evidencia de los bloques 1–4 está organizada por tópico dentro de `evidence/`; la del capstone (la intrusión completa) usa el esquema `E01`–`E12` en la raíz de `evidence/`, referenciado desde el informe de incidente y su anexo.

---

## Integración con el portafolio

Este es el **Proyecto 1** del mini-SOC AustralPay. Wazuh es el **consumidor final** del circuito de threat intel: consumirá IOCs enriquecidos desde n8n (Proyecto 4) vía la CDB list `malicious-ips.list` (ya preparada, con su regla de consulta lista para activar). La misma infraestructura es vigilada por Zabbix (Proyecto 2, disponibilidad/NOC), demostrando la operación completa **SOC + NOC**.

---

## Estado del proyecto

- [x] Wazuh single-node levantado y accesible
- [x] 3 agentes reportando + Sysmon en Windows
- [x] Telemetría de ataque generada y documentada
- [x] 5 reglas custom escritas, probadas y mapeadas a MITRE
- [x] Active Response, Vuln Detection y FIM activos
- [x] Retención de logs alineada a PCI-DSS Req. 10
- [x] Cadena de ataque completa ejecutada y documentada
- [x] Informe de incidente N1 redactado
- [x] Capa Navigator generada
- [x] CDB list de threat intel preparada (integración Proyecto 4)
