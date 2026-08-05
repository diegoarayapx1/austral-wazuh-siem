# Bloque 3A — Telemetría de ataque + MITRE ATT&CK (Wazuh SIEM)
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Wazuh SIEM · AustralPay Lab
Sesión: 3A

---

## Objetivo

Generar telemetría de ataque real desde una máquina atacante dedicada contra la infraestructura monitoreada de AustralPay, y verificar la detección del ruleset base de Wazuh, mapeando cada ataque a su técnica de MITRE ATT&CK. Los ataques cubren fuerza bruta SSH (T1110.001), fuerza bruta RDP (T1110.001) y ejecución de PowerShell sospechosa (T1059.001).

## Entorno

| Host | Rol | IP |
|---|---|---|
| `austral-siem` | Wazuh Manager | 192.168.1.50 |
| `austral-app` | Servidor de aplicación (target SSH) | 192.168.1.55 |
| `austral-ws01` | Estación Windows (target RDP + PowerShell) | 192.168.1.65 |
| `kali` | Máquina atacante dedicada (sin agente) | 192.168.1.100 |

| Componente | Detalle |
|---|---|
| Herramientas de ataque | hydra (SSH y RDP) |
| Verificación previa | nmap (confirmación de puertos 22 y 3389 abiertos) |
| Framework de referencia | MITRE ATT&CK |

## Qué se hizo

1. Despliegue de una máquina atacante dedicada (Kali, `192.168.1.100`) en la red bridged del laboratorio, **sin agente Wazuh**, para mantener una separación limpia entre el atacante y el entorno monitoreado.
2. Verificación de alcance y de puertos abiertos hacia los objetivos con nmap (SSH `22/tcp` en `austral-app`; RDP `3389/tcp` en `austral-ws01`).
3. **Fuerza bruta SSH (T1110.001)** contra `austral-app`: dos corridas de hydra, una con un diccionario que incluía una credencial válida (para observar el caso de acceso exitoso) y otra exclusivamente con credenciales inválidas (para observar la detección por umbral).
4. **Fuerza bruta RDP (T1110.001)** contra `austral-ws01`: hydra contra la cuenta `administrator`, generando eventos de fallo de logon `4625`.
5. **Ejecución de PowerShell sospechosa (T1059.001)** en `austral-ws01`: tres patrones de referencia (comando codificado en base64, ejecución con ventana oculta y bypass de política, y patrón de descarga en memoria), todos con contenido benigno para probar la detección sin ejecutar código malicioso.
6. Verificación de cada detección en el dashboard de Wazuh (Threat Hunting), registrando los `rule.id`, `rule.level` y el mapeo MITRE de las alertas.

## Detecciones observadas

| Ataque | Técnica MITRE | Reglas base (nivel) | Regla de correlación (nivel 10+) |
|---|---|---|---|
| Fuerza bruta SSH | T1110.001 | 5760, 5503, 5710 (nivel 5) | 5763, 5551, 2502 (nivel 10) |
| Acceso SSH exitoso | T1078 | login exitoso registrado | — |
| Fuerza bruta RDP | T1110.001 | 60122 (nivel 5) | 60204 (nivel 10) |
| PowerShell (comando codificado) | T1059.001 | — | 92057 (nivel 12) |
| PowerShell (proceso derivado) | T1059.001 | 92027 (nivel 4) | — |

Observaciones relevantes:

- **Fuerza bruta SSH:** una misma secuencia de fallos disparó tres reglas de correlación independientes (5763 vía decoder sshd, 5551 vía PAM, 2502 vía syslog genérico), todas de nivel 10. Esto ilustra la redundancia de detección del ruleset: múltiples reglas observando la misma actividad desde distintos parsers.
- **Acceso exitoso vs. detección por umbral:** cuando la credencial válida se encontró en los primeros intentos, el ataque concluyó antes de acumular suficientes fallos para disparar las reglas de correlación, quedando únicamente fallos individuales de nivel 5. Esto evidencia una limitación de las detecciones basadas en umbral frente a un acceso exitoso temprano — un candidato natural a regla personalizada (detección de acceso exitoso precedido de fallos desde la misma IP).
- **Fuerza bruta RDP:** los eventos `4625` registraron `logonType 3` (autenticación de red vía NTLM), consistente con una herramienta automatizada, y conservaron el nombre de la estación de trabajo de origen en el propio evento.
- **PowerShell:** el ruleset base detecta y mapea la actividad de PowerShell a T1059.001 de forma automática, incluyendo una regla de nivel 12 para el uso de comandos codificados en base64. La detección base, sin embargo, no evalúa el *contenido* de la línea de comandos: asigna un nivel informativo (4) a patrones de alto riesgo como la descarga y ejecución en memoria. Esta brecha de severidad es el punto de partida para las reglas personalizadas del bloque siguiente.

## Decisiones de diseño

- **Máquina atacante dedicada sin agente:** mantiene una separación limpia entre el atacante y el entorno monitoreado, con una IP de origen claramente ajena a los activos vigilados. Se descartó atacar desde un host ya monitoreado para no contaminar la telemetría con actividad ofensiva en un activo de la infraestructura.
- **Verificación previa con nmap:** confirmar el alcance y los puertos abiertos antes de lanzar los ataques evita diagnósticos erróneos sobre fallos de conectividad.
- **Contenido benigno en la PowerShell:** los comandos reproducen la *forma* de patrones de abuso (comando codificado, ventana oculta, descarga en memoria) sin ejecutar código dañino, permitiendo validar la detección de forma segura.
- **Dos corridas de fuerza bruta SSH:** generar tanto un acceso exitoso como una secuencia de solo fallos permite documentar ambos comportamientos de detección (correlación por umbral y evasión del umbral por acceso temprano).

## Evidencia

- `evidence/04-ataque-hydra.png` — Ejecución de hydra contra el servicio SSH de `austral-app`, mostrando los intentos y el resultado.
- `evidence/05-alerta-bruteforce.png` — Alerta de fuerza bruta SSH en el dashboard (regla 5763, nivel 10), con `rule.id`, `rule.level` y la IP de origen del atacante.

## Estado del checklist

- [x] Máquina atacante dedicada desplegada (sin agente)
- [x] Telemetría de ataque generada (fuerza bruta SSH, fuerza bruta RDP, PowerShell)
- [x] Detecciones base verificadas y `rule.id` registrados
- [x] Cada ataque mapeado a su técnica MITRE ATT&CK
- [x] Brecha de detección identificada (severidad de patrones en la línea de comandos de PowerShell)

## Próximo paso

Bloque 3B — desarrollo y prueba de reglas de detección personalizadas (`wazuh-logtest`): afinamiento de la detección de fuerza bruta (T1110) y una regla derivada que eleva la severidad de los patrones de abuso de PowerShell (T1059.001) identificados en este bloque.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
