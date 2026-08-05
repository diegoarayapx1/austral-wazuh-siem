# Timeline de la intrusión — INC-2026-001
**AustralPay es una organización ficticia de laboratorio.**

Timeline de la cadena de ataque ejecutada contra el laboratorio. Complementa el informe de incidente [`../incidents/INC-2026-001-cadena-intrusion.md`](../incidents/INC-2026-001-cadena-intrusion.md) y la capa [`navigator-layer.json`](./navigator-layer.json).

> **Zona horaria:** todos los timestamps en **UTC** (estándar de correlación SOC). Hora local del laboratorio (Chile) = UTC − 4.
> **Detección:** 🔴 = alerta generada por el SOC · ⚪ = acción silenciosa, reconstruida en la investigación.

## Secuencia (orden del atacante)

| Hora (UTC) | Host | Acción | Táctica (MITRE) | Detección | Ev. |
|---|---|---|---|---|---|
| 08:26 | austral-ws01 | Phishing: `wscript` ejecuta `.js` → lanza PowerShell | Initial Access (T1566 → T1059.005) | ⚪ contexto (92000, nivel 4) | E01 |
| 08:35 | austral-ws01 | Download-cradle de PowerShell (IEX + descarga) | Execution (T1059.001) | 🔴 **100210, nivel 12** ⚠️ *no investigada* | E02 |
| 08:50 | austral-ws01 | Persistencia: Run key `AustralUpdater` | Persistence (T1547.001) | ⚪ EID 13 sin alerta (gap de regla) | E03 |
| 09:29 | austral-ws01 | Robo de credencial desde `accesos-servidores.txt` | Credential Access (T1552.001) | ⚪ lectura no monitoreada | E04, E05 |
| 09:40 | austral-ws01 | Reconocimiento SSH a `austral-app:22` | Discovery (T1046) | ⚪ silencioso | E06 |
| 10:27 | austral-app | Login lateral con credencial robada (sin fallos) | Lateral Movement (T1078) | ⚪ 5715 nivel 3 (login de rutina) | E07 |
| **10:33** | austral-app | **[TRIGGER]** Brute force SSH exitoso | Lateral Movement (T1110.001) | 🔴 **100220, nivel 12** ← entrada de la investigación | E08, E09 |
| 10:57 | austral-app | Escalada de privilegios vía `sudo` | Privilege Escalation (T1548.003) | 🔴 5402, nivel 3 | E11 |
| 11:01 | austral-app | Modificación de `transacciones.csv` (datos de pago) | Impact (T1565.001) | 🔴 **550, nivel 7** (FIM whodata, con atribución) | E10, E12 |

**Ventana total:** 08:26 → 11:01 UTC (~2 h 35 min).
**Visibilidad del SOC:** 4 alertas dispersas en 2 hosts; 5 acciones silenciosas reconstruidas por pivoteo.

## Narrativa de descubrimiento

El analista entró por la mitad de la intrusión y expandió en ambas direcciones:

1. **Trigger (10:33, alerta 100220):** brute force de bajo volumen sobre una cuenta de pagos — se investiga, no se descarta.
2. **Pivote sobre el origen (.65) → 10:27:** aparece un login exitoso *previo* sin alerta. El acceso ya había ocurrido en silencio (hipótesis: credencial válida robada; orden causal exacto a confirmar por forense).
3. **Pivote hacia adelante → 10:57 / 11:01:** escalada a root e impacto sobre los datos de pago. El caso se eleva a **P1**.
4. **Pivote hacia atrás en `austral-ws01` → 09:40 / 09:29 / 08:50:** reconocimiento, robo de credencial (inferido) y persistencia, todo silencioso.
5. **Origen → 08:35 / 08:26:** el cradle (100210) y el phishing. Aquí emerge el **punto de detección desaprovechado**: la 100210 (nivel 12) se generó ~2.5 h antes y no fue investigada; atenderla habría detenido la cadena.

Ver análisis completo, alcance y recomendaciones en el [informe de incidente](../incidents/INC-2026-001-cadena-intrusion.md).
