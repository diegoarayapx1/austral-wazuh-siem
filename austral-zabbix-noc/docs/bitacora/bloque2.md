# Bloque 2 — Triggers, severidades, supresión y notificaciones (Zabbix NOC)
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Zabbix NOC · AustralPay Lab
Sesiones: 2A, 2B

---

## Objetivo

Construir la capa de alertado del NOC completa: desde la definición de qué constituye un problema (triggers y severidades justificadas por impacto de negocio), pasando por la reducción de ruido y la supresión de alertas redundantes, hasta la entrega de la notificación a una persona con el procedimiento de respuesta incluido.

## Entorno

| Componente | Detalle |
|---|---|
| Stack | Zabbix 7.0.29 LTS sobre Docker Compose |
| Server | `austral-noc` (`192.0.2.70`) |
| Hosts monitoreados | `austral-app`, `austral-db` (Linux), `austral-ws01` (Windows) |
| Telemetría de seguridad | Sysmon y agente SIEM en `austral-ws01` |
| Canal de notificación | Telegram Bot API (webhook nativo de Zabbix 7.0) |

## Estado inicial

Inventario de triggers heredados de las plantillas oficiales:

| Host | Triggers |
|---|---|
| `austral-app` | 30 |
| `austral-db` | 30 |
| `austral-ws01` | 72 |

Tres hallazgos que definieron el trabajo del bloque:

- Aproximadamente 50 de los 72 triggers de Windows provenían de la regla de descubrimiento de servicios, todos en severidad `Average`, incluyendo servicios sin relevancia operativa (cola de impresión, indexación de búsqueda, temas visuales, caché de fuentes, telemetría).
- Entre ellos, tres con valor real de seguridad: los servicios de Sysmon, del agente SIEM y del agente de monitoreo. Su detención no es un problema de disponibilidad ordinaria sino una posible manipulación de controles de seguridad.
- Ningún host tenía chequeo ICMP. La única detección de caída era indirecta (ausencia de datos del agente) y sin base válida para el cálculo de un SLA.

## Qué se hizo

### Capa de definición: qué constituye un problema

1. **Reducción de ruido** en `austral-ws01`: ampliación de la macro `{$SERVICE.NAME.NOT_MATCHES}` a nivel de host, sin modificar la plantilla, con 21 servicios cosméticos adicionales. Resultado: 21 triggers deshabilitados. Se mantuvieron monitoreados los servicios relevantes de seguridad y red del endpoint (registro de eventos, motor de filtrado base, firewall, caché DNS, servicios de red y de administración de cuentas).

2. **Triggers dedicados al stack de seguridad**: los cuatro servicios críticos se excluyeron del descubrimiento automático y se implementaron con items y triggers propios, en severidad `High`, con tags `mitre: T1562.001` y `scope: security`, y con el procedimiento de respuesta escrito en el campo de descripción del trigger.

| Trigger | Expresión |
|---|---|
| Sysmon detenido | `min(/austral-ws01/service.info["Sysmon64",state],#3)<>0` |
| Agente SIEM detenido | `min(/austral-ws01/service.info["WazuhSvc",state],#3)<>0` |
| Microsoft Defender detenido | `min(/austral-ws01/service.info["WinDefend",state],#3)<>0` |
| Agente de monitoreo detenido | `min(/austral-ws01/service.info["Zabbix Agent 2",state],#3)<>0` |

3. **Disponibilidad real por ICMP**: plantilla `ICMP Ping` aplicada a los tres hosts. Los hosts Linux respondieron de inmediato. El host Windows reportó 100% de pérdida por bloqueo del firewall local; se creó una regla de alcance mínimo (solo ICMP tipo 8, entrante, restringida a la dirección del servidor de monitoreo). Verificación antes y después: 100% de pérdida → 0%, RTT medio 0.274 ms.

4. **Supresión por dependencia**: declarada en los tres hosts, con el trigger de disponibilidad del agente dependiendo del trigger de ICMP. La cadena preexistente de la plantilla propaga la supresión en cascada a los tres triggers de caída.

| | Antes | Después |
|---|---|---|
| Alertas por caída total de host | 3 | 1 |
| Diagnóstico | ambiguo | causa raíz identificada |

### Capa de entrega: que la alerta llegue a una persona

Zabbix separa el alertado en cinco objetos independientes — evento, acción, operación, media type y media de usuario — con tres filtros de severidad y tiempo que operan por separado y deben alinearse: la condición de la acción, las severidades habilitadas en la media del usuario y su ventana horaria. Un desalineamiento en cualquiera produce silencio sin error visible. Existe un cuarto filtro implícito: los permisos de lectura del usuario sobre el host.

5. **Verificación de egress** desde dentro del contenedor del servidor hacia la API del canal (resolución DNS y handshake TLS con respuesta HTTP), previa a cualquier configuración. La verificación desde el host anfitrión no es válida: el contenedor tiene su propia pila de red y su propio `resolv.conf`.

6. **Media type** configurado, con el modo de formato de texto deshabilitado de forma deliberada, y validado de forma aislada mediante el botón Test antes de integrarlo con la cadena de acciones.

7. **Media de usuario** con ventana horaria completa y filtro de severidad restringido a Average, High y Disaster.

8. **Acción de trigger** con condición explícita `Trigger severity >= Average` y los tres bloques de operaciones configurados (Operations, Recovery operations, Update operations), con supresión de operaciones para problemas suprimidos activa.

9. **Plantillas de mensaje personalizadas** a nivel de media type:

```
Subject: [{EVENT.SEVERITY}] {HOST.NAME} - {EVENT.NAME}

INCIDENTE {EVENT.ID} | AustralPay NOC

Host: {HOST.NAME} ({HOST.IP})
Severidad: {EVENT.SEVERITY}
Inicio: {EVENT.DATE} {EVENT.TIME} UTC
Estado actual: {ITEM.LASTVALUE1}
Tags: {EVENT.TAGS}

ACCION ESPERADA:
{TRIGGER.DESCRIPTION}
```

`{TRIGGER.DESCRIPTION}` transporta el procedimiento de respuesta escrito dentro de cada trigger. El operador recibe la instrucción junto con el aviso, sin abrir la consola. `{EVENT.TAGS}` arrastra la taxonomía de clasificación, incluido el mapeo MITRE ATT&CK. La plantilla de recuperación incorpora `{EVENT.DURATION}`, insumo directo del cálculo de MTTR.

## Resultados de validación

Cuatro ciclos de caída y recuperación de `austral-ws01`. Todos los envíos registrados en el log de acciones en estado `Sent`, sin fallos ni duplicados.

Supresión por dependencia validada en las dos capas — dashboard (caída de 13 minutos, con visualización de problemas suprimidos habilitada) y notificación (caída controlada de 7 minutos):

| Métrica | Esperado | Resultado |
|---|---|---|
| Alertas notificadas durante la caída | 1 | 1 |
| Triggers de agente suprimidos | Sí | Sí, sin evento generado |
| Notificación de recuperación | Sí | Con duración calculada |

## Hallazgos técnicos

### La supresión por dependencia depende del orden de disparo

La conclusión inicial de la sesión 2A —que la dependencia previene el evento en origen— era correcta para la ventana observada (host apagado, trigger padre permanentemente en PROBLEM) pero **incompleta**. La sesión 2B documentó dos escenarios donde el mecanismo degrada:

**Recuperación escalonada.** Una dependencia retiene el evento del trigger hijo mientras el padre está en PROBLEM; no lo cancela. Si al resolverse el padre la condición del hijo sigue cumpliéndose, el evento se emite en ese momento. Medido: un host Windows responde a ICMP aproximadamente dos minutos antes de que el servicio de agente complete su arranque y reconecte. Esa diferencia es la ventana en que la dependencia deja de proteger.

**Host inestable.** En una prueba con el host recién reiniciado, el trigger hijo disparó 34 segundos antes que el padre. La dependencia no lo bloqueó porque el padre aún estaba en estado OK. Se generaron dos notificaciones donde el caso típico produce una.

Conclusión consolidada: la supresión es efectiva en el caso típico (caída limpia con host estable) y degrada en los bordes de recuperación y de inestabilidad. Es una propiedad del mecanismo, no un defecto de configuración.

### Semántica de evaluación de `nodata()`

Un trigger basado en `nodata()` sobre un ítem de tipo *active check* no se evalúa mientras falta el dato: el ítem no se refresca y no hay ciclo de evaluación. El trigger dispara **al restablecerse la recolección**, cuando la función mira hacia atrás y encuentra la ventana vacía.

Consecuencia operativa: una interrupción de servicio no declarada produce una ráfaga de alertas al recuperarse, no durante la caída. El operador recibe un problema que ya está resuelto. Esta es la razón operativa por la que existen las ventanas de mantenimiento.

### Comentario y acknowledge son acciones independientes

En el formulario de actualización de un problema, escribir un mensaje y marcar la casilla de reconocimiento son operaciones distintas. Solo la casilla genera el evento que dispara las operaciones de actualización. Un operador que comenta sin reconocer deja registro en la consola pero no notifica al resto del equipo, lo que anula el propósito de la operación.

Comportamiento adicional: al reabrir el formulario sobre un problema ya reconocido, la casilla aparece marcada reflejando el estado actual. Confirmar en esa condición ejecuta un *unacknowledge*.

### Desfase de zona horaria entre frontend y macros de evento

El frontend renderiza los tiempos en la zona horaria configurada; las macros de evento resuelven en UTC. En este entorno la diferencia es de cuatro horas, suficiente para producir un cambio de fecha entre ambas representaciones de un mismo evento.

Impacto: la correlación entre alertas de monitoreo y eventos del SIEM sobre una línea de tiempo compartida puede conducir a conclusiones erróneas si no se normaliza la referencia. Decisión adoptada: documentar la conversión en lugar de normalizar la zona horaria del frontend, y declarar explícitamente `UTC` en la plantilla del mensaje.

### Limitaciones de los overrides de descubrimiento en Zabbix 7.0

Un override configurado para elevar la severidad de los triggers generados por descubrimiento no surtió efecto pese a guardarse sin error. Dos limitaciones de la interfaz dificultaron el diagnóstico: el resumen de operaciones no refleja las acciones configuradas (muestra siempre el mismo texto derivado del campo de condición), y el botón de prueba no está disponible para reglas de descubrimiento sobre chequeos activos, porque es el agente quien inicia la conexión y el servidor no puede solicitar el valor a demanda.

Se aplicó criterio de proporcionalidad y se pivotó a triggers dedicados.

### El contexto de reinicio ya existe en la plantilla oficial

Se evaluó modificar el trigger de disponibilidad del agente para distinguir un arranque de máquina de una detención deliberada del servicio. La expresión no permite esa distinción: ambos escenarios producen el mismo valor. Se descartó también correlacionar con el tiempo de actividad del sistema dentro de la expresión, porque ese ítem lo reporta el propio agente y su último valor conocido no es confiable precisamente cuando el agente está caído.

La plantilla oficial de Windows ya provee un trigger de reinicio reciente en severidad Warning, por debajo del umbral de notificación. El contexto está disponible en consola para correlación manual, lo que convierte la distinción en una regla de procedimiento y no en un cambio de configuración.

## Decisiones de diseño

- **Macro a nivel de host en lugar de edición de plantilla**: modificar la plantilla afectaría a todo host futuro que la use. La sobrescritura selectiva deja la plantilla intacta y el cambio documentado en un solo lugar.
- **Triggers dedicados en lugar de override de descubrimiento**: un override es una optimización de escala. Con un host, los triggers explícitos son más claros y auditables, y aportan lo que el override no da: nombre propio, tag de técnica MITRE y descripción con procedimiento de respuesta.
- **Agente de monitoreo sin tag MITRE**: detener el agente puede ser evasión, pero es más probable que sea operación normal. Etiquetarlo como técnica de ataque sería inflar el mapeo. Severidad alta sí, porque la pérdida de visibilidad importa; atribución a técnica no.
- **Ventana de tres muestras en las expresiones de servicio**: con intervalo de un minuto, exige tres minutos de servicio caído antes de alertar. Evita el falso positivo por reinicio rutinario.
- **Regla de firewall restringida por origen en lugar de ICMP abierto**: se abrió el mínimo necesario. Efecto colateral: el host queda invisible al barrido de ping del resto de la red, lo que mitiga MITRE T1018 (Remote System Discovery) sin costo funcional.
- **Plantillas de mensaje a nivel de media type, no de operación**: la personalización pertenece al canal, no a una acción concreta. Una futura acción de escalamiento hereda el formato sin duplicar la plantilla.
- **Acción propia en lugar de la acción por defecto**: la acción preconfigurada no declara condiciones y notifica a un grupo. Definir la condición de severidad de forma explícita deja el criterio legible en la definición de la acción, además del filtro de la media. La redundancia es deliberada: los dos filtros se refuerzan en lugar de depender uno del otro.
- **Corte de severidad en Average**: criterio de mitigación de fatiga de alertas. Warning y por debajo permanecen en el dashboard; Average y superiores interrumpen a una persona. La reducción de ruido se hace en la capa de triggers; el filtro de la acción es el segundo colador, no el primero.
- **Formato de texto plano en el canal**: con marcado activo, un carácter de formato desbalanceado en un nombre de host o en el texto de un trigger provoca el rechazo del mensaje completo por parte de la API. El texto plano garantiza la entrega; la jerarquía visual se resuelve con mayúsculas y separación de bloques.
- **Media type de destino explícito en lugar de "todos los disponibles"**: previene la duplicación no intencionada si en el futuro se agrega un segundo canal al mismo usuario.
- **Verificación por capas antes de la prueba integrada**: egress del contenedor, media type aislado, media de usuario, y solo entonces la cadena completa. Esta secuencia permitió descartar tres capas de forma inmediata al aparecer un fallo posterior.
- **No afirmar en el mensaje una causa que el trigger no midió**: una alerta que declara un origen no verificado entrena al operador a asumir, y es peor que una alerta ruidosa.

## Consideración de seguridad sobre el canal de notificación

El uso de un servicio de mensajería externo como canal de alertado es válido en un entorno de laboratorio, pero no es transferible a producción en un contexto regulado. El envío de nombres de host, direcciones IP internas y descripciones de trigger a la API de un tercero constituye salida de información de infraestructura fuera del perímetro de control de la organización, incompatible con los requisitos de PCI-DSS aplicables a un entorno de pagos.

La ruta equivalente en producción es correo interno o una plataforma de gestión de guardias con rotación on-call, o bien un webhook hacia una plataforma de mensajería self-hosted.

Desde la perspectiva de detección, el mismo patrón es relevante en sentido inverso: la Bot API de Telegram es un canal de comando y control documentado en familias de malware, precisamente porque el tráfico sale por HTTPS hacia un dominio de buena reputación que rara vez se bloquea. El tráfico saliente generado por esta configuración es indistinguible en la capa de red del tráfico de un implante que use el mismo canal.

## Evidencia

| Archivo | Qué muestra |
|---|---|
| `01-icmp-firewall-antes-despues.png` | Ping desde el servidor de monitoreo al host Windows: 100% de pérdida antes de la regla de firewall, 0% después |
| `02-regla-firewall-icmp.png` | Regla de firewall con origen restringido, dirección entrante y acción permitir |
| `03-triggers-seguridad-high.png` | Los cuatro triggers del stack de seguridad en severidad alta con sus tags de clasificación |
| `04-items-seguridad-latest-data.png` | Los cuatro items de estado de servicio recolectando datos |
| `05-supresion-dependencia-icmp.png` | Vista de problemas con supresión habilitada: un único problema de disponibilidad visible tras la caída |
| `06-dependencia-configurada.png` | Declaración de la dependencia en el trigger hijo |
| `07-media-severidades.png` | Media de usuario con el filtro de severidad aplicado |
| `08-alerta-telegram-problema.png` | Notificación con formato personalizado y bloque de acción esperada |
| `09-ciclo-problema-recuperacion.png` | Ciclo problema y recuperación encadenado, con identificador compartido y duración calculada |
| `10-action-log.png` | Registro de acciones con los envíos en estado `Sent` |

## Estado del checklist

- [x] Inventario de triggers heredados de los tres hosts
- [x] Reducción de ruido en el host Windows
- [x] Triggers dedicados al stack de seguridad con mapeo MITRE
- [x] ICMP Ping aplicado a los tres hosts
- [x] Regla de firewall con alcance mínimo
- [x] Dependencias de supresión configuradas y validadas
- [x] Egress del contenedor de servidor verificado
- [x] Media type configurado, habilitado y validado de forma aislada
- [x] Media de usuario con filtro de severidad y ventana horaria
- [x] Acción con condición de severidad explícita
- [x] Operations, Recovery operations y Update operations configuradas
- [x] Plantillas de mensaje personalizadas con procedimiento embebido
- [x] Notificación de problema y recuperación validadas end-to-end
- [x] Registro de acciones sin envíos fallidos
- [ ] Notificación de acknowledge — problema abierto, ver limitaciones

## Limitaciones conocidas

| Ítem | Estado | Destino |
|---|---|---|
| Las operaciones de actualización no generan envío pese a configuración verificada y eventos de reconocimiento confirmados en el registro. Cuatro hipótesis formuladas y descartadas: ejecución sobre problema resuelto, conjunto vacío de destinatarios por antigüedad del evento, exclusión del usuario ejecutor, y error de marcado. No se registra ningún intento de envío, lo que apunta a falta de correspondencia entre el evento de actualización y la acción, no a un fallo de entrega | Abierto, sin causa raíz | Bloque 4A |
| El reinicio de un host genera notificación de severidad alta. El contexto de reinicio está disponible en consola pero no suprime la ráfaga | Documentado | Bloque 3B — ventanas de mantenimiento |
| El trigger de disponibilidad del agente Windows incorpora una descripción con mapeo MITRE ATT&CK T1562.001 que contradice su propio tag de clasificación, generando un falso positivo de seguridad en cada arranque del host. El mapeo corresponde a los triggers dedicados del stack de seguridad, no a este | Pendiente de corrección | — |
| Desfase de zona horaria entre frontend y macros de evento | Documentado, mitigado en plantilla | No se corrige |
| Descripciones heredadas de plantillas oficiales en idioma distinto al de las descripciones propias | Cosmético | Opcional |
| Uso de disco del host Windows sobre el umbral de alerta (19.3 GB totales) | Aceptado | Limitación declarada del laboratorio |
| El host por defecto del propio servidor de monitoreo se auto-reactiva | Documentado en Bloque 1 | Decisión de scope |

## Próximo paso

Bloque 3 — módulo Services/SLA: definición de servicio, umbral de disponibilidad y reporte, sobre la base de datos que aportan los items ICMP configurados en este bloque. Continúa con auto-discovery, escalamiento de alertas y ventanas de mantenimiento.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
