# Mapeo MITRE ATT&CK — Reglas de detección personalizadas (Wazuh SIEM)
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Wazuh SIEM · AustralPay Lab
Documento: mapeo de las reglas de detección personalizadas a MITRE ATT&CK

---

## Propósito

Este documento relaciona cada regla de detección personalizada desarrollada en el proyecto con la técnica de MITRE ATT&CK que detecta, el procedimiento concreto con que se validó, y la regla base de Wazuh sobre la que se apoya. Además, mapea la **cadena de intrusión completa** ejercitada en el incidente INC-2026-001 (Bloque 5) a MITRE, con su estado de detección. Sirve como referencia de cobertura de detección y como base para la capa de MITRE ATT&CK Navigator (`navigator-layer.json`).

Todas las reglas residen en `/var/ossec/etc/rules/local_rules.xml` en el manager y usan identificadores en el rango personalizado (100000+).

---

## Resumen de cobertura

| Regla | Nivel | Táctica | Técnica | Regla base | Validación |
|---|---|---|---|---|---|
| 100210 | 12 | Execution | T1059.001 — PowerShell | 92027 (`if_sid`) | Pipeline real |
| 100211 | 0 | — (tuning de FP) | — | 92213 (`if_sid`) | Pipeline real |
| 100212 | 0 | — (tuning de FP) | — | 92213 (`if_sid`) | Pipeline real |
| 100220 | 12 | Credential Access / Defense Evasion / Persistence / Privilege Escalation / Initial Access | T1110.001 — Password Guessing + T1078 — Valid Accounts | 5715 (`if_sid`) + 5760 (`if_matched_sid`) | logtest + pipeline real |
| 100230 | 12 | Defense Evasion / Persistence / Privilege Escalation / Initial Access | T1078 — Valid Accounts | 5715 (`if_sid`) | logtest + pipeline real |

---

## Detalle por regla

### 100210 — PowerShell download cradle (T1059.001)

**Táctica:** Execution
**Técnica:** T1059.001 — Command and Scripting Interpreter: PowerShell

**Qué detecta:** ejecución de un download-cradle de PowerShell — descarga y ejecución de código en memoria mediante la combinación de `IEX`/`Invoke-Expression` con una primitiva de descarga remota (`DownloadString`, `Net.WebClient`, `Invoke-WebRequest`, etc.).

**Lógica de detección:** regla hija de la 92027 (PowerShell derivando PowerShell). Exige dos condiciones combinadas con AND sobre `win.eventdata.commandLine`: presencia de un intérprete en memoria (`IEX`) **y** de una descarga remota. El requisito de ambas condiciones eleva la precisión y reduce falsos positivos frente a matchear cualquiera de las dos por separado.

**Aporte sobre el ruleset base:** el ruleset base detecta el patrón pero le asigna nivel informativo (4). Esta regla lo eleva a nivel 12 (alto), reflejando el riesgo real de una descarga y ejecución en memoria.

**Procedimiento de validación:** ejecución en el endpoint Windows monitoreado de un comando PowerShell con la forma de un download-cradle (contenido benigno), verificando en el pipeline real que la alerta se promueve a la 100210 nivel 12 mapeada a T1059.001.

---

### 100211 — Tuning de falso positivo (sin técnica MITRE)

**Táctica / Técnica:** ninguna. Es una regla de tuning, no de detección.

**Qué hace:** silencia un falso positivo crítico del ruleset base. La regla base 92213 ("Executable file dropped in folder commonly used by malware", nivel 15) dispara sobre el archivo `__PSScriptPolicyTest_*.ps1` que PowerShell crea en cada arranque en el directorio temporal para evaluar la política de ejecución. Es un artefacto benigno que generaba un crítico falso en cada ejecución de PowerShell.

**Lógica:** regla hija de la 92213 que matchea el prefijo fijo `__PSScriptPolicyTest_` en `win.eventdata.targetFilename` y asigna nivel 0 (evaluada pero sin generar alerta).

**Decisión de diseño:** la excepción se ancla al prefijo específico del artefacto benigno, no a "cualquier archivo en el directorio temporal". Un dropper real en la misma carpeta sigue disparando la 92213 en nivel 15. La regla de tuning es tan específica como el falso positivo que corrige, sin abrir un hueco de detección.

**No lleva mapeo MITRE** porque suprimir un falso positivo no es detectar una técnica; adjuntarle una técnica sería incorrecto.

---

### 100212 — Tuning de falso positivo: cleanmgr / WimProvider.dll (sin técnica MITRE)

**Táctica / Técnica:** ninguna. Regla de tuning, no de detección.

**Qué hace:** silencia un segundo falso positivo de la regla base 92213. Durante la ejecución del incidente INC-2026-001 se observó que `cleanmgr.exe` (Liberador de espacio en disco de Windows) suelta `WimProvider.dll` — un componente legítimo del sistema — en su carpeta temporal, disparando la 92213 (nivel 15) de forma repetida. Caso de fatiga de alertas.

**Lógica:** regla hija de la 92213 anclada a **dos** campos combinados: `win.eventdata.image` = `cleanmgr.exe` **y** `win.eventdata.targetFilename` = `WimProvider.dll`. Asigna nivel 0.

**Decisión de diseño:** el anclaje a los dos campos mantiene la especificidad. Un `cleanmgr.exe` soltando cualquier *otra* DLL sigue disparando la 92213 en nivel 15, de modo que no se abre hueco al side-loading vía binarios legítimos. Consistente con el criterio de la 100211.

**No lleva mapeo MITRE**, igual que la 100211.

---

### 100220 — Brute force SSH exitoso (T1110.001 + T1078)

**Tácticas:** Credential Access, Defense Evasion, Persistence, Privilege Escalation, Initial Access
**Técnicas:** T1110.001 — Brute Force: Password Guessing · T1078 — Valid Accounts

**Qué detecta:** un login SSH exitoso precedido de múltiples fallos desde la misma IP — es decir, un ataque de fuerza bruta que **tuvo éxito**. Este es el caso que las reglas de correlación base por umbral pueden dejar pasar: si el atacante acierta la credencial antes de acumular suficientes fallos, no se cruza el umbral de las reglas composite y solo quedan fallos sueltos de bajo nivel.

**Lógica de detección (correlación stateful):** regla que dispara sobre el login exitoso (`if_sid` sobre la 5715) únicamente si estuvo precedido de fallos (`if_matched_sid` sobre la 5760) desde la misma IP (`same_srcip`), con un umbral de 4 fallos en 120 segundos.

**Por qué mapea a dos técnicas:** el ataque es fuerza bruta (T1110.001), pero un brute force exitoso resulta en el uso de una cuenta válida (T1078), por lo que hereda las tácticas asociadas al abuso de credenciales legítimas.

**Procedimiento de validación:** validada por dos métodos. (1) `wazuh-logtest`: cuatro líneas de fallo seguidas de una de éxito desde la misma IP promueven a la 100220. (2) Pipeline real: fuerza bruta con hydra desde la máquina atacante, generando los fallos y el acceso exitoso, verificando la alerta con la IP de origen, el usuario objetivo y los fallos precedentes registrados en el propio evento.

---

### 100230 — Acceso con credencial válida desde IP no confiable (T1078)

**Tácticas:** Defense Evasion, Persistence, Privilege Escalation, Initial Access
**Técnica:** T1078 — Valid Accounts

**Qué detecta:** un login SSH exitoso originado desde una IP sin rol legítimo en la operación — un acceso con credencial válida desde un origen ilegítimo. Cubre el caso del atacante que ya **posee** la credencial (robada, filtrada, reutilizada) y entra directamente, sin fuerza bruta previa, escenario que la 100220 no detecta porque no hay fallos que correlacionar.

**Lógica de detección:** regla hija del login exitoso (`if_sid` sobre la 5715) condicionada a que el `srcip` corresponda a un origen no confiable. En el laboratorio ese origen es la IP de la máquina atacante.

**Complementariedad con la 100220:** ambas reglas detectan el mismo objetivo del adversario (acceso con cuenta válida) por dos caminos distintos — la 100220 detecta la credencial **adivinada** (fuerza bruta exitosa), la 100230 detecta la credencial **usada directamente** desde un origen ilegítimo. Juntas cubren tanto el acceso adivinado como el acceso directo.

**Límite conocido (expuesto en INC-2026-001):** la 100230 depende de que el origen esté clasificado como *no confiable*. En el incidente, el movimiento lateral con credencial robada (variante 6B) provino de un host **interno y whitelisteado** (`austral-ws01`); para la regla ese origen es confiable, por lo que **no disparó**. Este es un blind spot real: la detección basada en IP de origen no cubre al atacante que ya está dentro de un host de confianza. La mitigación es detección basada en comportamiento (ver informe INC-2026-001, §8).

**Nota de diseño (detección vs prevención):** esta regla es un control de **detección**, no de prevención. La primera línea de defensa ante un origen no confiable es una ACL o regla de firewall que bloquee ese acceso. La regla del SIEM opera como capa de detección dentro de una estrategia de defensa en profundidad: si el control preventivo falla, está mal configurado, o el acceso llega por otra vía, el evento igual se registra y se alerta. En producción, la lista de orígenes no confiables se implementaría como una CDB list actualizable —el mismo mecanismo por el cual el SIEM consume indicadores de compromiso desde fuentes de threat intel externas.

**Procedimiento de validación:** validada por dos métodos. (1) `wazuh-logtest`: un login exitoso desde la IP no confiable dispara la 100230; un login exitoso desde una IP confiable NO la dispara (solo la regla base de éxito), confirmando la especificidad. (2) Pipeline real: acceso SSH directo desde la máquina atacante, verificando la alerta nivel 12 mapeada a T1078 sin fuerza bruta previa.

---

## Cobertura de la cadena de intrusión INC-2026-001

Esta sección mapea la **intrusión completa** ejercitada en el Bloque 5 (informe `caso-INC-2026-001.md`) a MITRE ATT&CK, más allá de las reglas custom. Incluye técnicas detectadas por reglas base y técnicas que quedaron como **gap** de detección. Es la fuente directa de la capa `navigator-layer.json` y empata 1:1 con ella.

**Estado de detección:** `DETECTADO` = generó alerta · `CONTEXTO` = evento de bajo nivel sin valor de alerta · `GAP` = no detectado.

| Técnica | Nombre | Táctica | Estado | Mecanismo / motivo |
|---|---|---|---|---|
| T1566 | Phishing | Initial Access | CONTEXTO | Vector de entrada (`.js` señuelo); visible solo vía la ejecución downstream |
| T1059.005 | Command and Scripting Interpreter: Visual Basic | Execution | CONTEXTO | `wscript` ejecuta el `.js` → regla base 92000, nivel 4 |
| T1059.001 | Command and Scripting Interpreter: PowerShell | Execution | DETECTADO | Regla custom **100210**, nivel 12 |
| T1547.001 | Boot or Logon Autostart Execution: Registry Run Keys | Persistence | GAP | Sysmon EID 13 registra la Run key; ninguna regla Wazuh la promueve (gap de regla, no de sensor) |
| T1552.001 | Unsecured Credentials: Credentials In Files | Credential Access | GAP | Lectura del `.txt` de credenciales; leer un archivo no genera evento (punto ciego total) |
| T1110.001 | Brute Force: Password Guessing | Credential Access | DETECTADO | Regla custom **100220**, nivel 12 |
| T1046 | Network Service Discovery | Discovery | GAP | Reconocimiento con herramienta nativa (`Test-NetConnection`); gap inherente |
| T1018 | Remote System Discovery | Discovery | GAP | Barrido interno del segmento; gap inherente |
| T1021.004 | Remote Services: SSH | Lateral Movement | PARCIAL | Variante brute force (6A) detectada por 100220; variante con credencial robada (6B) evade toda detección |
| T1078 | Valid Accounts | Defense Evasion | GAP | Credencial robada usada desde host interno confiable; la 100230 no aplica (origen whitelisteado). Base del blind spot 6B |
| T1548.003 | Abuse Elevation Control Mechanism: Sudo and Sudo Caching | Privilege Escalation | DETECTADO | Regla base **5402**, nivel 3 (control auditado) |
| T1005 | Data from Local System | Collection | GAP | Lectura del `transacciones.csv`; la lectura/exfiltración no deja rastro |
| T1565.001 | Data Manipulation: Stored Data Manipulation | Impact | DETECTADO | FIM whodata, regla base **550**, nivel 7, con atribución completa |

**Balance de cobertura:** 13 técnicas en 10 tácticas → **4 detectadas**, **3 contexto/parcial**, **6 gaps**. La lectura del patrón: el atacante fue detectable al ejecutar código (100210), forzar credenciales (100220), escalar por sudo (5402) y modificar datos bajo FIM (550); fue **invisible** al persistir, robar credenciales en texto plano, reconocer, y moverse lateralmente con credencial válida desde un host de confianza. La detección madura requiere análisis de comportamiento, no solo firmas y reputación de IP.

---

## Técnicas cubiertas (vista para Navigator)

Tabla completa que alimenta la capa `navigator-layer.json`. Combina las técnicas detectadas por reglas custom y las de la cadena INC-2026-001.

| Técnica | Nombre | Táctica (lane Navigator) | Estado | Regla / mecanismo |
|---|---|---|---|---|
| T1566 | Phishing | initial-access | CONTEXTO | 92000 (downstream) |
| T1059.005 | Visual Basic | execution | CONTEXTO | 92000 base |
| T1059.001 | PowerShell | execution | DETECTADO | 100210 (custom) |
| T1547.001 | Registry Run Keys | persistence | GAP | Sysmon EID 13 sin regla |
| T1552.001 | Credentials In Files | credential-access | GAP | lectura no monitoreada |
| T1110.001 | Password Guessing | credential-access | DETECTADO | 100220 (custom) |
| T1046 | Network Service Discovery | discovery | GAP | herramienta nativa |
| T1018 | Remote System Discovery | discovery | GAP | herramienta nativa |
| T1021.004 | Remote Services: SSH | lateral-movement | PARCIAL | 100220 (6A) / gap (6B) |
| T1078 | Valid Accounts | defense-evasion | GAP | 100230 no aplica (origen confiable) |
| T1548.003 | Sudo and Sudo Caching | privilege-escalation | DETECTADO | 5402 base |
| T1005 | Data from Local System | collection | GAP | lectura no deja rastro |
| T1565.001 | Stored Data Manipulation | impact | DETECTADO | 550 base (FIM whodata) |

> La capa exportable de Navigator está en `attack-chain/navigator-layer.json`. Colores: verde = detectado, ámbar = contexto/parcial, rojo = gap.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
