# Runbook de respuesta por trigger — NOC AustralPay

> **AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.** Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.

**Ámbito:** operador N1 del NOC.
**Plataforma:** Zabbix 7.0 LTS.
**Última revisión:** 2026-08-17.

---

## Cómo usar este documento

Cada alerta que llega por Telegram incluye una línea `Runbook:` que apunta aquí. Busca el trigger por su nombre y sigue las secciones en orden.

**No saltes la sección de verificación.** Existe porque en varios de estos triggers la lectura literal de la alerta lleva a la conclusión equivocada, y actuar sobre un diagnóstico incorrecto alarga el incidente en vez de resolverlo.

**El criterio de escalamiento no es negociable.** Si se cumple el tiempo indicado sin resolución, se escala. Insistir más allá de ese límite no es diligencia: es retrasar a quien sí puede resolverlo.

---

## Principios generales

Cuatro reglas que aplican a cualquier alerta, antes de abrir la sección específica.

**1. Verifica que el instrumento esté encendido antes de creerle al silencio.**
La ausencia de alertas puede significar que todo está bien o que la recolección está caída. Si el panel se ve inusualmente limpio, comprueba que hay datos frescos antes de asumir normalidad.

**2. Distingue "no responde" de "no reporta".**
Son dos fallas distintas con dos causas distintas. Un equipo puede estar vivo y no responder a una prueba concreta.

**3. Un problema abierto necesita un dato nuevo para cerrarse.**
Los triggers se resuelven cuando llega un valor que satisface la condición de recuperación. Si la recolección se interrumpe con un problema abierto, ese problema permanece abierto indefinidamente aunque la falla real haya terminado.

**4. Toma nota de la hora en que asumes el incidente.**
El acknowledge no es un trámite: detiene la cadena de escalamiento y deja constancia auditable de quién tomó el caso. Un incidente sin acknowledge escala al turno superior aunque alguien lo esté trabajando.

---

## `Unavailable by ICMP ping` — dispositivo de red

**Severidad:** High

### Qué significa
El servidor de monitoreo dejó de recibir respuesta a las pruebas de alcance contra el equipo. Se exigen tres sondeos fallidos consecutivos antes de disparar, de modo que no es un paquete perdido.

### Impacto
Potencial pérdida de conectividad del segmento servido por el equipo. En el caso del router de borde, incomunicación entre sedes.

### Verificación

1. **¿Sigue llegando telemetría SNMP?**
   `Monitoring → Latest data`, filtrar por el host, revisar la columna `Last check` de un item de intervalo corto (uptime, estado de interfaz).
   - **Hay datos frescos (segundos):** el equipo está vivo. Esto **no** es una caída — es un fallo de camino, filtro o política que afecta solo a ICMP. Continúa en el paso 2.
   - **No hay datos nuevos:** el equipo está efectivamente inalcanzable. Salta al paso 3.

2. **Equipo vivo sin respuesta a ping.** Causas en orden de frecuencia:
   - Lista de control de acceso que descarta ICMP desde el servidor de monitoreo
   - Política de seguridad aplicada recientemente sobre la interfaz de gestión
   - Cambio de ruta que rompe el camino de vuelta
   Revisa si hubo cambios en el equipo en la última hora. Escala a Redes indicando explícitamente que **el equipo responde SNMP pero no ICMP** — ese dato ahorra al siguiente nivel la mitad del diagnóstico.

3. **Equipo inalcanzable.** Verifica alcance desde un segundo origen distinto del servidor de monitoreo. Si tampoco responde, confirma el estado de energía y de la plataforma que lo aloja.

### Acción
- Fallo de camino o filtro: escalar a Redes con el detalle del paso 2.
- Caída confirmada: escalar a Redes. Registrar la hora del último dato recibido — marca el inicio real del incidente, que es anterior a la alerta.

### Escalamiento
**10 minutos** sin recuperación ni causa identificada → Redes.

### Nota
Un equipo que responde a gestión pero no al plano de datos es un **fallo parcial**. Es el escenario más peligroso: el monitoreo lo ve "medio vivo" y el operador tiende a clasificarlo como caída total o como falso positivo. No es ninguna de las dos.

---

## `Interface <nombre>: Link down`

**Severidad:** Average

### Qué significa
El estado operativo de la interfaz pasó de activo a caído. El trigger exige una **transición**, no un estado: una interfaz que ya estaba caída al incorporarse al monitoreo no genera alerta.

### Impacto
Depende de la interfaz. Un enlace hacia un segmento de servicio interrumpe tráfico productivo; un puerto no utilizado, ninguno. Consulta la descripción de la interfaz en la propia alerta — se propaga desde la configuración del equipo.

### Verificación

1. **¿Hay también una alerta de indisponibilidad del equipo?**
   Si la hay, la caída de enlace es consecuencia y no causa. Atiende la indisponibilidad primero.
   Las dependencias configuradas suprimen esta alerta cuando el equipo entero cae, de modo que si la ves sola, el equipo está vivo.

2. **En el equipo:** `show ip interface brief` y `show interfaces <interfaz>`.
   - `administratively down` → desactivación administrativa. Alguien ejecutó un cambio.
   - `down/down` con estado administrativo activo → pérdida de portadora. Falla de medio físico, del equipo del otro extremo, o de transceptor.

3. **Revisa si hubo cambios recientes.** Una desactivación administrativa no ocurre sola.

### Acción
- Desactivación administrativa no planificada: escalar a Redes. **No reviertas por tu cuenta** — puede ser parte de un cambio en curso.
- Pérdida de portadora: escalar a Redes con el resultado de `show interfaces`, incluyendo contadores de errores.

### Escalamiento
**15 minutos** → Redes.

### Notas
- Un cambio aplicado solo a la configuración en ejecución **se revierte al reiniciar el equipo, sin alertar a nadie**. Si un enlace vuelve a estado activo sin intervención, comprueba si hubo un reinicio de por medio.
- **No deshabilites este trigger para silenciarlo durante un incidente.** Deshabilitar un trigger destruye el evento activo, y al rehabilitarlo el problema no reaparece: la expresión exige una transición que ya ocurrió. El resultado es un fallo real invisible en consola. Para silenciar sin perder visibilidad, usa una ventana de mantenimiento.

---

## `has been restarted` — equipo o servidor

**Severidad:** Warning

### Qué significa
El contador de tiempo de actividad se reinició. El equipo arrancó hace menos de diez minutos.

### Impacto
En sí mismo, ninguno: el equipo ya está operativo. El riesgo está en lo que el reinicio pudo llevarse.

### Verificación

1. **¿Estaba planificado?** Consulta ventanas de mantenimiento activas y el registro de cambios. Si lo estaba, cierra el incidente con la referencia.

2. **Si no estaba planificado**, verifica en este orden:
   - **Configuración:** ¿sobrevivieron los cambios recientes? Los aplicados solo a la configuración en ejecución se perdieron.
   - **Recolección:** ¿sigue llegando telemetría? Ciertos parámetros de identidad del subsistema de monitoreo se regeneran al arrancar e invalidan las credenciales establecidas, con síntoma de fallo de autenticación pese a credenciales correctas.
   - **Causa:** revisa el registro del equipo para determinar si fue un reinicio ordenado o una pérdida de energía.

### Acción
- Planificado: cerrar con referencia al cambio.
- No planificado: escalar a Redes o a Sistemas según el equipo, indicando si la configuración y la recolección sobrevivieron.

### Escalamiento
Reinicio no planificado → escalar **siempre**, sin espera. No es urgente, pero no debe quedar sin investigar.

---

## `No SNMP data collection`

**Severidad:** Warning

### Qué significa
El servidor de monitoreo dejó de obtener datos por SNMP, sin que el equipo esté necesariamente caído.

### Impacto
**Pérdida de visibilidad.** El equipo puede estar fallando ahora mismo y no habría alerta. Se trata con más urgencia de lo que sugiere su severidad.

### Verificación

1. **¿Responde a ping?** Si no, el problema es de alcance, no de SNMP. Atiende `Unavailable by ICMP ping`.
2. **Si responde a ping pero no entrega SNMP:**
   - ¿Hubo un reinicio reciente? Ver el trigger anterior — es la causa más frecuente.
   - ¿Se modificó la lista de control de acceso del subsistema SNMP?
   - ¿Se rotaron credenciales sin actualizarlas en el monitoreo?

### Acción
Escalar a Redes indicando que el equipo responde pero no entrega telemetría. Mientras dure, **el equipo está sin supervisión efectiva** — decláralo así en el escalamiento.

### Escalamiento
**15 minutos** → Redes.

---

## `High ICMP ping loss`

**Severidad:** Warning

### Qué significa
Pérdida parcial de paquetes por encima del umbral. El equipo responde, pero de forma degradada.

### Impacto
Degradación de servicio, no interrupción. Latencia y reintentos en aplicaciones sensibles.

### Verificación

1. **¿Es transitorio?** Una ráfaga de pérdida durante el arranque de varios sistemas simultáneos es esperable y se resuelve sola en minutos.
2. **¿Es sostenido?** Revisa el gráfico del item de pérdida en la última hora. Un patrón sostenido apunta a saturación de enlace, error de medio o problema en el equipo del otro extremo.
3. **Contadores de error** en la interfaz correspondiente.

### Acción
- Transitorio y ya recuperado: registrar y cerrar.
- Sostenido: escalar a Redes con el gráfico y los contadores de error.

### Escalamiento
**20 minutos** sostenido → Redes.

### Nota
Pérdida parcial e indisponibilidad total son dos triggers distintos con dos respuestas distintas. Un enlace con 40% de pérdida sigue "arriba" en cualquier panel de estado y para el usuario está roto.

---

## `Zabbix agent is not available`

**Severidad:** High

### Qué significa
El agente instalado en el servidor dejó de reportar durante el intervalo definido.

### Impacto
Pérdida de visibilidad del servidor. No implica que el servicio esté caído.

### Verificación

1. **¿El host responde a ping?**
   - No → el servidor está caído o inalcanzable. Trátalo como caída de host.
   - Sí → el servidor está vivo y el problema es del agente.
2. **Agente detenido en un servidor vivo:** puede ser reinicio del servicio, actualización en curso, o interrupción deliberada.
3. **Revisa si hay otras alertas del mismo host.** Un agente caído junto a otras señales anómalas merece más atención que uno aislado.

### Acción
- Servidor caído: escalar a Sistemas.
- Agente caído en servidor vivo: escalar a Sistemas para reinicio del servicio.

### Escalamiento
**15 minutos** → Sistemas.

### Nota
La detención del agente de monitoreo también es una técnica de evasión: interrumpir la recolección antes de actuar reduce la probabilidad de detección. No se etiqueta como incidente de seguridad por defecto, porque la explicación operativa es mucho más frecuente. Pero si coincide con otras señales anómalas del mismo host, se notifica al equipo de seguridad además del escalamiento normal.

---

## `Space is critically low` — sistema de archivos

**Severidad:** Average

### Qué significa
El uso del sistema de archivos superó el umbral crítico.

### Impacto
Riesgo de interrupción de servicio. Una base de datos sin espacio deja de aceptar escrituras; un sistema sin espacio en la partición raíz deja de funcionar de forma impredecible.

### Verificación

1. **¿Cuánto queda y a qué velocidad se consume?** Revisa el gráfico de la última semana. Un consumo lineal permite estimar cuándo se agota; un salto brusco indica un evento concreto.
2. **Qué está ocupando el espacio.** Los sospechosos habituales son registros sin rotación, respaldos acumulados y archivos temporales.

### Acción
- Escalar a Sistemas con la tasa de crecimiento y la estimación de tiempo restante.
- **No elimines archivos por cuenta propia.** Un registro borrado puede ser evidencia necesaria para otra investigación.

### Escalamiento
Escalar **siempre**, sin espera. La urgencia la fija el tiempo restante estimado, no la severidad del trigger.

---

## `Base de datos no disponible`

**Severidad:** Warning

### Qué significa
La comprobación de disponibilidad del servicio de base de datos falló.

### Impacto
**Directo sobre el servicio de procesamiento de pagos.** Consume presupuesto de error del acuerdo de nivel de servicio desde el primer minuto.

### Verificación

1. **¿El servidor responde a ping?** Si no, es caída de host, no de base de datos.
2. **¿El puerto acepta conexión?** Determina si el servicio está detenido o sobrecargado.
3. **Revisa el espacio en disco del servidor** — es una causa frecuente de detención del motor.

### Acción
Escalar a Sistemas de inmediato, indicando el impacto sobre el servicio de pagos.

### Escalamiento
**5 minutos.** Es el trigger de menor tolerancia del runbook por su impacto directo sobre el servicio de negocio.

### Nota
La comprobación verifica alcance a nivel de transporte, no que el motor acepte consultas ni que responda en tiempo razonable. Un motor vivo pero saturado puede pasar esta verificación mientras el servicio está degradado para el usuario.

---

## Matriz de escalamiento

| Trigger | Severidad | Espera | Destino |
|---|---|---|---|
| `Base de datos no disponible` | Warning | 5 min | Sistemas |
| `Unavailable by ICMP ping` | High | 10 min | Redes |
| `Zabbix agent is not available` | High | 15 min | Sistemas |
| `Link down` | Average | 15 min | Redes |
| `No SNMP data collection` | Warning | 15 min | Redes |
| `High ICMP ping loss` | Warning | 20 min sostenido | Redes |
| `has been restarted` (no planificado) | Warning | Inmediato | Según equipo |
| `Space is critically low` | Average | Inmediato | Sistemas |

La severidad del trigger no determina el orden de atención. El impacto sobre el servicio, sí.

---

## Anexo — Lecturas engañosas de la consola

Casos verificados en este entorno donde la interfaz no dice lo que aparenta.

**Un host en verde puede no estar entregando datos.**
El indicador de disponibilidad refleja el estado del transporte de monitoreo, no la llegada de métricas. Verde con telemetría detenida es un estado posible. Verifica siempre la marca de tiempo del último dato.

**`Last check` no significa "cuándo se consultó".**
Significa "cuándo se almacenó el último valor". Los items con descarte de valores repetidos solo almacenan cuando el valor cambia o cuando vence su intervalo de refresco, que puede ser de horas. Un item estático sin actualizar en doce horas puede estar funcionando perfectamente.

**Una interfaz permanentemente caída no genera alerta, y es correcto.**
El trigger exige una transición. Una interfaz que nunca estuvo activa no produce ninguna. Ausencia de alerta no equivale a fallo de detección.

**Un problema puede quedar abierto indefinidamente si la recolección se interrumpe.**
Si el monitoreo se detiene con un problema activo, ese problema permanece hasta que llegue un dato que satisfaga la condición de recuperación. Un incidente de duración anómala frente a la falla real casi siempre es esto.

**El monitoreo también se alerta a sí mismo.**
Las alertas del propio servidor de monitoreo indican presión sobre la recolección, no fallas de la infraestructura vigilada. Ignorarlas significa que las lecturas siguientes dejan de ser confiables sin que nadie lo advierta.

**La ausencia de datos y la supresión por dependencia se ven idénticas.**
Cuando un trigger dependiente no aparece, puede ser porque la dependencia lo contuvo o porque nunca recibió el dato que lo dispara. Distinguirlo exige revisar el historial del item, no la lista de problemas.

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.*
