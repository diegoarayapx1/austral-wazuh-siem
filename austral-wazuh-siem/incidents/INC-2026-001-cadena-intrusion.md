# INC-2026-001 — Compromiso de host y acceso no autorizado a datos de pago
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Este informe es un ejercicio de análisis; ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.**

Informe de incidente — SOC N1 · Proyecto AustralPay (Wazuh SIEM)

> **Zona horaria:** todos los timestamps en **UTC** (estándar de correlación SOC). El laboratorio operó en hora de Chile (UTC−4); hora local = UTC − 4 (p. ej., 10:33 UTC = 06:33 local). *Nota de revisión: una versión previa de trabajo registró las horas en local; se normalizaron todas a UTC para consistencia y correlación cross-zona.*

---

## 1. Clasificación y severidad

| Campo | Valor |
|---|---|
| ID de incidente | INC-2026-001 |
| Analista | DiegoAraya — SOC N1 |
| Fecha de detección | 2026-08-04, 10:33 UTC |
| Fecha del informe | 2026-08-04 |
| Tipo de incidente | Intrusión multi-etapa con acceso no autorizado y modificación de datos |
| **Severidad inicial (triage)** | **P3 — Media** · brute force SSH de bajo volumen (3 fallos + éxito), origen interno |
| **Severidad final (post-investigación)** | **P1 — Crítica** · reclasificada a las 11:01 UTC tras confirmar escalada a root e integridad de datos de pago comprometida |
| Datos regulados involucrados | Sí — datos de pago (contexto PCI-DSS) |
| Hosts afectados | `austral-ws01` (192.168.1.65), `austral-app` (192.168.1.55) |
| Cuenta comprometida | `soporte-pagos` (servicio de pagos), escalada a root |
| Estado | **Escalado a N2** — contención pendiente |

**Justificación de la reclasificación P3 → P1:** la señal de entrada (brute force de bajo volumen) no reflejaba el alcance real del incidente. La investigación reveló un acceso previo con credencial válida, persistencia, escalada a root e impacto sobre `transacciones.csv`, elevando el caso a crítico. La evaluación inicial fue razonable con la información disponible en el momento del triage; reclasificar al alza cuando la evidencia lo exige es parte del criterio de análisis, no una corrección de un error.

---

## 2. Resumen ejecutivo

El 2026-08-04, entre las **08:26 y 11:01 UTC**, un actor no autorizado comprometió la estación `austral-ws01` mediante un archivo de phishing que ejecutó un *download-cradle* de PowerShell (alerta 100210, nivel 12, 08:35 UTC). Desde ese punto de apoyo estableció persistencia, obtuvo credenciales de la cuenta `soporte-pagos` almacenadas en texto plano, y las utilizó para acceder al servidor de pagos `austral-app`. Una vez dentro, escaló privilegios a root mediante una configuración `sudo` permisiva y modificó el archivo de transacciones `transacciones.csv` (alerta FIM 550, 11:01 UTC).

El SOC detectó la intrusión a través de **4 alertas dispersas en 2 hosts**; las etapas de robo de credenciales, reconocimiento y movimiento lateral inicial no generaron alertas y fueron reconstruidas durante la investigación. El caso entró a la cola de triage como un brute force SSH de bajo volumen (P3) y se reclasificó a **P1 (Crítico)** al confirmarse el impacto sobre datos de pago.

**Hallazgo crítico de proceso:** la alerta 100210 (nivel 12) se generó ~2.5 horas antes del impacto y no fue investigada. Su seguimiento oportuno habría expuesto la cadena y permitido contenerla antes de que alcanzara el servidor de pagos (*missed detection opportunity*).

Se recomienda contención inmediata de ambos hosts, rotación de la credencial `soporte-pagos` y escalamiento a N2/forense para determinar si hubo exfiltración de datos.

---

## 3. Timeline del incidente

**Convención:** la columna *Detección* indica si el SOC vio el evento en tiempo real (🔴 alerta) o si se reconstruyó durante la investigación (⚪ silencioso). Orden cronológico del atacante; la narrativa de descubrimiento (cómo el analista lo recorrió) se detalla debajo.

| Hora (UTC) | Host | Acción del atacante | Táctica (MITRE) | Detección | Ev. |
|---|---|---|---|---|---|
| 08:26 | austral-ws01 | Entrega de phishing: `wscript` ejecuta `.js` → lanza PowerShell | Initial Access (T1566 → T1059.005) | ⚪ contexto (92000, nivel 4) | E01 |
| 08:35 | austral-ws01 | Ejecución del download-cradle (IEX + descarga remota) | Execution (T1059.001) | 🔴 **100210, nivel 12** ⚠️ *no investigada* | E02 |
| 08:50 | austral-ws01 | Persistencia: Run key `AustralUpdater` | Persistence (T1547.001) | ⚪ silencioso — EID 13 sin alerta (gap de regla) | E03 |
| 09:29 | austral-ws01 | Robo de credencial desde `accesos-servidores.txt` | Credential Access (T1552.001) | ⚪ silencioso — lectura no monitoreada | E04, E05 |
| 09:40 | austral-ws01 | Discovery: sondeo SSH a `austral-app:22` | Discovery (T1046) | ⚪ silencioso | E06 |
| 10:27 | austral-app | Login lateral con credencial robada (sin fallos) | Lateral Movement (T1078) | ⚪ silencioso — 5715 nivel 3 (login de rutina) | E07 |
| **10:33** | austral-app | **[TRIGGER]** Brute force SSH exitoso (3 fallos + éxito) | Lateral Movement (T1110.001) | 🔴 **100220, nivel 12** ← *entrada de la investigación* | E08, E09 |
| 10:57 | austral-app | Escalada de privilegios vía `sudo` | Privilege Escalation (T1548.003) | 🔴 5402, nivel 3 | E11 |
| 11:01 | austral-app | Modificación de `transacciones.csv` (datos de pago) | Impact (T1565) | 🔴 **550, nivel 7** (FIM whodata, con atribución) | E10, E12 |

**Ventana total del incidente:** 08:26 → 11:01 UTC (~2 h 35 min).
**Visibilidad del SOC:** 4 alertas (100210, 100220, 5402, 550) dispersas en 2 hosts; 5 acciones silenciosas reconstruidas por pivoteo.

### 3.1 Narrativa de descubrimiento

El analista no recorrió el incidente en el orden del atacante, sino en el orden en que la investigación lo reveló, **entrando por la mitad de la intrusión y expandiendo hacia atrás y hacia adelante**:

1. **Entrada — 10:33, alerta 100220.** Brute force SSH de bajo volumen (3 fallos + éxito) contra `soporte-pagos` desde el host interno .65. El bajo volumen no justifica un P1 automático, pero el patrón fallo→éxito sobre una cuenta de servicio de pagos amerita seguimiento, no descarte.
2. **Pivote sobre el origen (.65) → 10:27.** Aparece un login exitoso *previo* (evento 5715, sin alerta), anterior al brute force. Esto reorienta la investigación: el acceso ya se había producido en silencio. **Hipótesis de trabajo:** el atacante ya poseía credenciales válidas antes del brute force; el orden causal exacto entre ambos accesos no es determinable con la telemetría disponible a nivel N1 y se escala para correlación forense.
3. **Pivote hacia adelante → 10:57 y 11:01.** El acceso silencioso desembocó en escalada a root e **impacto sobre los datos de pago**. En este punto el caso se eleva a **P1**.
4. **Pivote hacia atrás en `austral-ws01` → 09:40 / 09:29 / 08:50.** Discovery, robo de credencial (inferido por el uso posterior) y persistencia — todo silencioso.
5. **Origen → 08:35 (cradle) y 08:26 (phishing).** Aquí emerge el **punto de detección desaprovechado** (ver §3.2).

### 3.2 Punto de detección desaprovechado (*missed detection opportunity*)

La alerta **100210** (nivel 12, ejecución de PowerShell) se generó a las 08:35 UTC pero no fue investigada. En ausencia de actividad ruidosa posterior en el mismo host, y sin un proceso de correlación entre hosts, una alerta de severidad alta aislada quedó sin seguimiento — comportamiento consistente con un SOC de baja madurez. Esto constituye el punto de detección desaprovechado del incidente: el cradle era un indicador **inequívoco** de ejecución maliciosa (no un falso positivo ambiguo), y su investigación oportuna habría expuesto la persistencia (08:50) y detenido la cadena ~2.5 horas antes del impacto sobre los datos de pago. Se traduce en la recomendación #1 (§8).

---

## 4. Análisis y alcance (blast radius)

**Hosts comprometidos:** `austral-ws01` (punto de entrada / paciente cero) y `austral-app` (objetivo, servidor de pagos).

**Cuenta comprometida:** `soporte-pagos`, escalada a **root** en `austral-app`. Con privilegios de root, el alcance de acceso del atacante **no está limitado a la actividad observada**: root puede leer, modificar o eliminar cualquier archivo del host, incluidos logs. Esto amplía el alcance potencial más allá de lo evidenciado.

**Impacto sobre datos — clasificado por nivel de certeza:**

| Categoría | Detalle | Base |
|---|---|---|
| **Confirmado** | Acceso de lectura al archivo completo de transacciones + modificación de integridad (inserción de un registro) | Evidencia directa: `cat` del CSV + alerta FIM 550 con atribución `soporte-pagos → sudo → root` |
| **No determinable** | Exfiltración de los datos leídos | La lectura de archivos no genera telemetría de exfiltración; no se puede confirmar ni descartar con los datos disponibles |
| **Asumido (precaución)** | Exfiltración potencial y acceso a otros datos/credenciales del host | Consecuencia de privilegios root + acceso de lectura confirmado a datos regulados. Se asume el peor caso hasta descarte forense |

**Datos regulados:** el archivo afectado contiene datos de pago (contexto PCI-DSS). Independiente de si la exfiltración se confirma, la modificación de integridad de datos de transacciones es en sí un impacto reportable que requiere validación forense y evaluación de obligaciones de notificación.

---

## 5. IOCs y artefactos

Un IOC (*Indicator of Compromise*) es un dato observable que delata la intrusión. Su valor: permite al N2 cazar el mismo actor en otros hosts y cargar bloqueos/detecciones para reincidencia.

| Tipo | Indicador | Contexto |
|---|---|---|
| Host / IP | `192.168.1.65` (`austral-ws01`) | Origen del movimiento lateral; estación paciente cero |
| Cuenta | `soporte-pagos` | Cuenta comprometida, escalada a root |
| Archivo | `Factura-Pendiente-AustralPay.js` (Downloads) | Señuelo de phishing / vector de ejecución |
| Persistencia | Run key `HKCU\...\CurrentVersion\Run\AustralUpdater` | Mecanismo de persistencia — **eliminar en remediación** |
| Patrón de comando | `IEX (New-Object Net.WebClient).DownloadString(...)` | Firma del download-cradle |
| Archivo sensible expuesto | `accesos-servidores.txt` (Documents) | Fuente de la credencial robada — **eliminar** |
| Archivo afectado | `/opt/austral-app/data/transacciones.csv` | Datos de pago modificados |

**Nota de alcance:** al ser un laboratorio aislado, los IOCs son internos (IPs privadas, sin hashes de malware real ni dominios C2 externos). En un incidente real, aquí irían además hashes de binarios, dominios/URLs de C2 e IPs externas de origen. Se documenta la limitación para no sobre-representar el alcance.

---

## 6. Cobertura de detección y gaps

De las 8 tácticas de la intrusión, el stack detectó 4 y no vio 4:

| Etapa | Táctica | Detección | Gap |
|---|---|---|---|
| Execution | 100210 nivel 12 ✅ | — |
| Lateral (brute force) | 100220 nivel 12 ✅ | — |
| Privilege Escalation | 5402 ✅ | — |
| Impact | 550 nivel 7 ✅ (FIM whodata) | — |
| Persistence | ❌ | Sysmon generó EID 13; ninguna regla lo promueve (gap de **regla**, no de sensor) |
| Credential Access | ❌ | Lectura de archivo no monitoreada (punto ciego total) |
| Discovery | ❌ | Reconocimiento con herramientas nativas (gap inherente) |
| Lateral (cred robada) | ❌ | Login válido desde host confiable — indistinguible de actividad legítima |

**Lectura del patrón:** el atacante fue **invisible mientras abusó de confianza** (credencial válida, host interno) y **solo se volvió detectable al cruzar controles auditados** (sudo, FIM). La detección basada únicamente en "IP de origen confiable" es evadible; la prioridad de mejora es la detección **basada en comportamiento**.

---

## 7. Acciones inmediatas / handoff a N2

*Registro de contención requerida para traspaso. Ejecución y confirmación a cargo de N2.*

- [ ] Aislar `austral-ws01` (192.168.1.65) y `austral-app` (192.168.1.55) de la red.
- [ ] Deshabilitar la cuenta `soporte-pagos` y rotar su credencial.
- [ ] Revocar la regla sudoers `soporte-pagos ALL=(ALL) ALL` en `austral-app`.
- [ ] Eliminar el artefacto de persistencia: Run key `HKCU\...\Run\AustralUpdater` en `austral-ws01`.
- [ ] Eliminar el archivo de credenciales en claro `accesos-servidores.txt`.
- [ ] Preservar evidencia (imágenes de disco/memoria de ambos hosts) antes de remediar, para forense.

---

## 8. Recomendaciones de detección y hardening

*Derivadas de la visibilidad de triage: qué falló en la detección y por qué la cadena avanzó desapercibida. Cada recomendación se ancla a la etapa del incidente que habría detenido o revelado.*

| # | Hallazgo (qué falló) | Recomendación | Etapa que remedia | Tipo | Prioridad |
|---|---|---|---|---|---|
| 1 | Alerta 100210 (nivel 12) sin investigar por ~2.5h (*missed detection opportunity*) | Toda alerta de nivel ≥12 genera ticket de investigación obligatorio, independiente de actividad posterior en el host | Execution (08:35) — habría cortado la cadena antes del impacto | Proceso | **Crítica** |
| 2 | Movimiento lateral con credencial válida desde host interno indistinguible de un login legítimo | Detección por comportamiento: alertar sobre primer acceso de una cuenta a un host nuevo, u horarios/patrones anómalos — no confiar solo en la IP de origen | Movimiento lateral con credencial válida (10:27) | Técnica | **Alta** |
| 3 | Persistencia por Run key registrada por Sysmon (EID 13) pero sin alerta en Wazuh | Crear regla que promueva EID 13 sobre claves de autostart (`Run`/`RunOnce`) con allowlist de entradas legítimas | Persistence (08:50) | Técnica (regla) | **Alta** |
| 4 | Robo de credencial en texto plano completamente invisible (leer no es evento) | Control **preventivo** (no detectable): prohibir credenciales en texto plano, desplegar gestor de credenciales, escaneo periódico de secretos en disco | Credential Access (09:29) | Política + Técnica | **Alta** |
| 5 | Cuenta de operador con `sudo` total sobre el servidor de pagos | Mínimo privilegio: restringir sudo a comandos específicos; eliminar `ALL=(ALL) ALL` para cuentas de operación | Priv. Escalation (10:57) | Política | **Media** |
| 6 | Reconocimiento interno no detectable con herramientas nativas | Segmentación de red: limitar qué hosts puede alcanzar una estación de operaciones (no debería llegar al SSH del servidor de pagos) | Discovery (09:40) + Lateral | Técnica (arquitectura) | **Media** |
| 7 | Vector de entrada por phishing sobre usuario sin capacitación | Capacitación de concientización + filtrado de adjuntos ejecutables (`.js`) en correo | Initial Access (08:26) | Política | **Media** |

**Priorización:** las recomendaciones 1–4 atacan los puntos donde la cadena avanzó *desapercibida* — mayor retorno de detección. La #1 es crítica porque es la única que, sola, habría detenido *este* incidente completo: la detección técnica ya existía (la alerta se generó); lo que faltó fue el **proceso** para actuar sobre ella. Es el hallazgo central del caso: la mejor detección no sirve sin el proceso operativo para responder a ella.

---

## 9. Estado y siguiente paso

**Estado:** escalado a N2 para respuesta y a N3/forense para determinación de exfiltración. Contención pendiente de ejecución (ver §7). Incidente **abierto**.

**Rol del informe:** este es el paquete de traspaso del N1 al N2 — triage, reconstrucción de la cadena, alcance delimitado y recomendaciones. La contención, erradicación y cierre corresponden a N2/N3 conforme a sus atribuciones. El N1 no cierra el incidente; lo escala con contexto.

**Analista N1:** DiegoAraya. **Traspaso:** 2026-08-04, 11:15 UTC.

---

## Evidencias

Las capturas que respaldan cada punto del timeline (referenciadas E01–E12 en la columna *Ev.*) se documentan en el anexo visual **`Anexo-Evidencias-INC-2026-001.pdf`**, con cada evidencia mapeada a su técnica, mecanismo de detección y archivo de repositorio.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
