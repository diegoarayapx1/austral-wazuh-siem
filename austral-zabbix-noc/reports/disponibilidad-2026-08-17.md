# Informe de disponibilidad — AustralPay

> **AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.** Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.

**Período declarado:** 2026-08-17, 11:40:33 – 17:00:00 (America/Santiago, UTC−4)
**Duración de la ventana:** 5h 19m 27s
**Emitido:** 2026-08-17
**Alcance:** servicios de negocio y de seguridad definidos en el modelo de servicio de Zabbix, más la capa de red monitoreada.

---

## 1. Resumen ejecutivo

Durante la ventana declarada, **los servicios de negocio cumplieron su objetivo sin consumir error budget**: `Procesamiento de Pagos`, `Base de datos transaccional` y `Servidor de aplicación` registraron 100% de disponibilidad contra un SLO de 99,9%. Los servicios de seguridad registraron 100% contra un SLO de 99%.

Ese resultado, sin embargo, **no debe leerse como que la plataforma estuvo disponible.** El dispositivo de borde de red estuvo inalcanzable **1h 35m** dentro de la misma ventana —de los cuales 1h 14m por una falla espontánea no provocada— y el indicador de cumplimiento no se movió, porque el modelo de servicio no incluye la capa de red entre las dependencias del servicio de pagos.

**La conclusión operativa de este período no es un número de disponibilidad, sino una brecha de modelado:** el indicador que la organización usa para declarar cumplimiento no cubre un componente cuya caída interrumpe el servicio de cara al comercio. Se documenta como hallazgo principal y se acompaña de la corrección recomendada en la sección 8.

Adicionalmente, se declara que **el período calendario no es reportable** en este entorno: la semana cerrada 2026-08-09 / 2026-08-15 arroja 84,81% para el servicio de pagos, cifra dominada por indisponibilidad aparente producida por apagones de la propia plataforma de monitoreo. El detalle y el motivo de no corregir esa cifra están en la sección 7.

---

## 2. Alcance y período declarado

### 2.1 Por qué no se reporta sobre período calendario

Un reporte de disponibilidad se emite normalmente sobre período calendario —mes o semana cerrada— porque el usuario del servicio sufre el calendario, no la ventana de trabajo del equipo.

En este entorno esa premisa no se sostiene, por un mecanismo verificado durante el período de operación anterior:

Cuando el servidor de monitoreo se detiene, **el sesgo sobre el indicador no es simétrico**:

- Un problema **ya abierto** no requiere datos para permanecer abierto; requiere datos para cerrarse. Su duración se contabiliza durante todo el apagón. → **Se sobrecuenta.**
- Un problema que **habría ocurrido** durante el apagón no se detecta nunca, porque ningún trigger puede evaluarse sin datos. → **Se subcuenta a cero.**

No es ruido aleatorio que se compense en el promedio: es sesgo direccional. En el caso documentado con mayor detalle —el enlace `Gi0/1`— de 6d 10h 09m contabilizados como indisponibilidad, **6d 08h 12m (98,7%) correspondieron a período sin monitoreo activo**.

En consecuencia, este informe se emite sobre **período declarado**: una ventana acotada en la que se verificó, con dos fuentes independientes, que el sistema de monitoreo estuvo operativo de extremo a extremo.

### 2.2 Ventanas del período

| Ventana | Rango (hora local) | Duración | Tratamiento |
|---|---|---|---|
| **V1 — Arranque de plataforma** | 11:22:51 → 11:40:33 | 17m 42s | **Excluida del cómputo.** Encendido secuencial del entorno, no operación |
| **V2 — Operación** | 11:40:33 → 17:00:00 | **5h 19m 27s** | **Ventana sobre la que se reporta** |

**La exclusión de V1 se declara aquí, con su rango exacto y su justificación.** Esa es la única forma admisible de excluir tiempo de un cómputo de cumplimiento: escrita en el reporte y verificable por el lector. La alternativa —retirar el tramo desde la configuración de la herramienta— produce un número mejor sin dejar rastro auditable, y por eso se descarta como práctica. La discusión completa está en la sección 9.

### 2.3 Verificación de la ventana

La afirmación "el monitoreo estuvo activo durante toda V2" se sostiene sobre dos fuentes que no comparten mecanismo:

| Fuente | Evidencia |
|---|---|
| Historial de ítems | Serie continua de datos de sondeo ICMP a intervalo de 1 minuto durante toda la ventana, sin huecos atribuibles al servidor |
| Registro del servidor de monitoreo | Secuencia completa de eventos de backoff y recuperación de sondeo SNMP, correspondiente uno a uno con los eventos registrados en la consola |

La correspondencia entre ambas fuentes es exacta en los seis puntos de transición de la ventana. El método está detallado en el Anexo A.

---

## 3. Cumplimiento por servicio

### 3.1 Servicios de negocio — SLO 99,9%

| Servicio | SLO | SLI del período declarado | Indisponibilidad | Error budget consumido |
|---|---|---|---|---|
| Procesamiento de Pagos | 99,9% | **100%** | 0s | **0%** |
| Base de datos transaccional | 99,9% | **100%** | 0s | **0%** |
| Servidor de aplicación | 99,9% | **100%** | 0s | **0%** |

### 3.2 Servicios de seguridad — SLO 99%

| Servicio | SLO | SLI del período declarado | Error budget consumido |
|---|---|---|---|
| Cobertura de controles de seguridad | 99% | **100%** | **0%** |
| Endpoint austral-ws01 | 99% | **100%** | **0%** |

### 3.3 Nota sobre el error budget de una ventana corta

El error budget de un SLO de 99,9% sobre esta ventana es de **19,17 segundos**. Sobre el SLO de 99% de los servicios de seguridad, **3m 12s**.

Esa cifra no es un objetivo operativo realista y no debe presentarse como tal. **Un SLO se define sobre el período del contrato —típicamente mensual— y evaluarlo sobre una ventana de cinco horas lo vuelve inalcanzable por construcción:** cualquier incidente único de más de veinte segundos agota el presupuesto completo del período.

El error budget se incluye aquí porque es la unidad correcta para dimensionar el impacto relativo de los incidentes de la sección 5, no como métrica de cumplimiento contractual.

---

## 4. Hallazgo principal: el indicador no cubre el borde de red

### 4.1 El hecho

El modelo de servicio configurado es:

```
AustralPay
├── Procesamiento de Pagos                (SLO 99,9%)
│   ├── Base de datos transaccional
│   └── Servidor de aplicación
└── Cobertura de controles de seguridad   (SLO 99%)
    └── Endpoint austral-ws01
```

El dispositivo de borde de red, `austral-net`, **está monitoreado pero no está modelado**: no figura como hijo ni como dependencia de ningún servicio del árbol.

Durante V2, `austral-net` acumuló **1h 35m de indisponibilidad medida**, y ningún indicador de cumplimiento registró variación alguna.

### 4.2 Causa: deriva del modelo de servicio

La omisión no fue un descuido de diseño. Es un desfase cronológico verificable:

| Fecha | Hecho |
|---|---|
| 2026-08-07 | Se define el modelo de servicio y se crean los objetos SLA |
| 2026-08-11 | Se incorpora `austral-net` al monitoreo |

**El modelo de servicio se construyó antes de que el dispositivo existiera en el inventario, y no se revisó al incorporarlo.**

Esto es más relevante que un error puntual, porque describe un mecanismo que se repite en operación real: el modelo de servicio se define una vez, al inicio del proyecto, y la infraestructura sigue creciendo. Cada incorporación posterior que no dispara una revisión del modelo **amplía la distancia entre lo que se monitorea y lo que se mide**.

> **La deriva del modelo de servicio es silenciosa por construcción.** Un componente no modelado no genera error, no aparece en ningún listado de faltantes y no degrada ningún indicador. Su única manifestación es un indicador que permanece verde cuando no debería, y eso solo se detecta si alguien contrasta una caída conocida contra el indicador — que es exactamente cómo se detectó aquí.

El control que corresponde no es corregir este caso, sino **incorporar la revisión del modelo de servicio al procedimiento de alta de infraestructura**, de modo que ningún componente entre al monitoreo sin una decisión explícita sobre a qué servicio pertenece o por qué no pertenece a ninguno.

### 4.3 Por qué importa

AustralPay es una pasarela de procesamiento de pagos. El borde de red es el punto por el que los comercios alcanzan la plataforma. **Un borde inalcanzable durante 74 minutos consecutivos significa, en términos de negocio, servicio interrumpido durante 74 minutos** — mientras el reporte de cumplimiento declara 100%.

### 4.4 Formulación general

> **El SLI no mide lo que la infraestructura hace. Mide lo que el modelo de servicio declara que importa.** Un indicador en verde no prueba que el servicio estuvo disponible: prueba que los componentes modelados lo estuvieron. Toda dependencia no declarada es un punto ciego con apariencia de cumplimiento.

### 4.5 Un segundo caso de la misma familia

El servicio `Endpoint austral-ws01` reporta 100% en el período. Su definición lo asocia a problemas etiquetados con alcance de **seguridad**, no de disponibilidad. La estación `austral-ws01` estuvo inalcanzable 16 minutos durante V1 y el servicio no lo registró.

**El cálculo es correcto para lo que el servicio mide.** El defecto es de denominación: un servicio llamado "Endpoint austral-ws01" que reporta 100% comunica al lector "la estación estuvo disponible", cuando lo que mide es "no hubo problemas de seguridad en la estación".

| Servicio | Mecanismo | Clase de defecto |
|---|---|---|
| `austral-net` | La dependencia real no está en el árbol | Omisión de modelado |
| `Endpoint austral-ws01` | Mide correctamente, se denomina de forma engañosa | Ambigüedad semántica |

Ambos producen el mismo efecto sobre quien lee el reporte: **un número verde que no significa lo que su etiqueta sugiere.**

---

## 5. Registro de incidentes del período declarado

Todos los incidentes del período afectaron a `austral-net`. Se clasifican por naturaleza porque miden cosas distintas: los provocados evalúan la instrumentación, los espontáneos evalúan la plataforma.

| # | Inicio | Fin | Duración | Severidad | Evento | Naturaleza |
|---|---|---|---|---|---|---|
| 1 | 11:54:36 | 12:07:36 | 13m 00s | High | Dispositivo inalcanzable por ICMP | **Provocado** — detención de nodo para prueba de dependencia |
| 2 | 12:07:36 | 12:08:36 | 1m 00s | Warning | Alta pérdida de paquetes ICMP | Secuela del rearranque del incidente 1 |
| 3 | 12:08:03 | 12:20:02 | 11m 59s | Warning | Dispositivo reiniciado (uptime < 10m) | Informativo |
| 4 | 12:11:36 | 12:19:36 | 8m 00s | High | Dispositivo inalcanzable por ICMP | **Provocado** — filtrado selectivo de ICMP para aislar variable |
| 5 | 13:50:39 | 15:19:02 | 1h 28m 23s | Warning | Dispositivo reiniciado (uptime < 10m) | Informativo |
| 6 | **13:55:36** | **15:09:36** | **1h 14m 00s** | **High** | **Dispositivo inalcanzable por ICMP** | **Espontáneo — sin intervención** |

### 5.1 Indisponibilidad agregada de `austral-net`

| Concepto | Tiempo | Disponibilidad sobre V2 |
|---|---|---|
| Indisponibilidad total medida | 1h 35m 00s | **70,26%** |
| — atribuible a pruebas provocadas | 21m 00s | — |
| — atribuible a falla espontánea de plataforma | 1h 14m 00s | **76,84%** |

Si `austral-net` estuviera modelado bajo `Procesamiento de Pagos` con un SLO de 99,9%, la falla espontánea por sí sola habría consumido **232 veces** el error budget de la ventana.

### 5.2 Incidente 6 — caída espontánea

Es el único incidente del proyecto **no provocado por el operador y atendido sin presencia de operador**. Por esa razón es el único cuyos tiempos de respuesta reflejan una condición operativa realista.

Secuencia reconstruida con fuentes cruzadas:

| Hora local | Hito | Fuente |
|---|---|---|
| 13:50:20 | El dispositivo deja de responder — reinicio espontáneo | Registro del servidor |
| 13:50:36 | Conectividad restablecida brevemente | Registro del servidor |
| 13:50:39 | Trigger de reinicio detectado (Warning) | Consola |
| 13:52:36 | Última muestra de sondeo satisfactoria | Historial de ítem |
| 13:53:36 | Primera muestra de sondeo fallida | Historial de ítem |
| 13:53:50 | Primer error de red SNMP registrado | Registro del servidor |
| 13:54:35 | Sondeo SNMP suspendido por backoff | Registro del servidor |
| **13:55:36** | **Detección — evento de indisponibilidad creado** | Consola |
| 13:55:39 | Notificación entregada, paso 1 | Registro de acciones |
| 14:00:39 | Notificación entregada, paso 2 | Registro de acciones |
| — | **Sin acknowledge en todo el incidente** | Registro de acciones |
| 15:09:20 | Sondeo SNMP rehabilitado — conectividad restituida | Registro del servidor |
| **15:09:36** | **Recuperación — evento cerrado** | Consola |
| 15:09:38 | Notificación de recuperación entregada | Registro de acciones |
| 15:19:02 | Cierre del trigger de reinicio | Consola |

---

## 6. Tiempos de respuesta

### 6.1 Los dos denominadores

Los tiempos de respuesta se reportan desde dos orígenes distintos, y la diferencia entre ambos es información, no redundancia:

- **Desde la detección** — mide al equipo de operación. Es lo que el equipo controla.
- **Desde la falla real** — mide lo que experimentó el usuario del servicio. Incluye la latencia de detección.

Reportar únicamente desde la detección oculta sistemáticamente el tramo de detección y comunica a la dirección una realidad más benigna que la real.

### 6.2 Valores del período

| Incidente | Naturaleza | MTTD | MTTA | MTTR desde detección | MTTR desde falla real |
|---|---|---|---|---|---|
| 1 | Provocado | 2m 36s | 2m 37s | 13m 00s | 15m 36s |
| 4 | Provocado | — | — | 8m 00s | — |
| **6** | **Espontáneo** | **2m 00s – 3m 00s** | **No aplica** | **1h 14m 00s** | **~1h 17m** |

### 6.3 Lecturas

**La latencia de detección es una decisión de diseño, no una deficiencia.** El trigger de indisponibilidad exige tres sondeos fallidos consecutivos sobre un ítem de intervalo de 1 minuto. Ese umbral se eligió deliberadamente para no generar alertas por pérdida aislada de paquetes, y su costo es entre dos y tres minutos de latencia. Es un intercambio explícito entre falsos positivos y tiempo de detección, y se documenta como tal.

**El MTTD del incidente 6 se reporta como intervalo, no como valor puntual.** La falla ocurrió en algún punto entre la última muestra satisfactoria y la primera fallida, separadas por un intervalo de sondeo. **La resolución temporal de la medición está acotada por la frecuencia de sondeo**, y expresar el dato con precisión de segundos sería atribuir a la medición una exactitud que no tiene. Los incidentes 1 y 4 sí admiten valor puntual porque la hora de falla se conoce de forma independiente: la provocó el operador.

**El MTTA del incidente 6 no aplica: el incidente nunca fue reconocido.** El servicio se restituyó mediante una intervención sobre la infraestructura que no quedó vinculada a la alerta. El registro resultante muestra un incidente detectado, notificado, escalado y resuelto, **sin trazabilidad de quién respondió ni cuándo**.

En operación real esto es una brecha de coordinación de turno, no un detalle administrativo: sin acknowledge, ningún otro operador sabe que el incidente tiene dueño, y la escalada continúa avisando a niveles superiores de algo que ya está siendo atendido. **Un turno sin cobertura convirtió una falla de tres minutos en una interrupción de setenta y cuatro.**

---

## 7. Contraste con el período calendario

Se incluye la lectura calendario para dejar constancia de la magnitud de la distorsión, no como cifra de cumplimiento.

**Semana cerrada 2026-08-09 / 2026-08-15 — SLO 99,9%:**

| Servicio | SLI calendario | Indisponibilidad implícita | Error budget consumido |
|---|---|---|---|
| Base de datos transaccional | 84,8137% | 25h 30m 46s | 152× el presupuesto |
| Procesamiento de Pagos | 84,8134% | 25h 30m 48s | 152× el presupuesto |
| Servidor de aplicación | 98,8443% | 1h 56m 29s | 11,6× el presupuesto |

**Estas cifras no son reportables**, y el motivo por el que no se corrigen es tan relevante como el dato:

La descomposición entre indisponibilidad real e indisponibilidad aparente por apagón del monitoreo **está probada para el enlace `Gi0/1` (98,7% aparente) pero no fue medida para los servicios de negocio de esa semana.** Aplicar a estos últimos el porcentaje obtenido para otro objeto sería extrapolación sin base.

Ante esa incertidumbre, la salida correcta es **declarar la cifra como no interpretable, no ajustarla**. Un número corregido con un factor no medido es menos honesto que un número malo declarado como tal.

### 7.1 El período elige el número

La misma medición del mismo dispositivo, leída sobre ventanas distintas:

| Ventana de lectura | Disponibilidad de `austral-net` |
|---|---|
| Últimas 3 horas al cierre | **57,78%** |
| Ventana declarada V2 (5h 19m) | **70,26%** |
| V2 descontando pruebas provocadas | **76,84%** |

Ninguna de las tres es falsa. **Un porcentaje de disponibilidad sin período declarado no es un dato: es una elección.** Todo indicador de este informe se acompaña de su ventana exacta por esa razón.

### 7.2 Un indicador de período abierto no es un número

Durante la elaboración de este informe, el mismo servicio y el mismo período —la semana en curso, aún abierta— arrojaron tres valores distintos:

| Lectura | Valor |
|---|---|
| Primera | 98,7266% |
| Segunda, minutos después | 98,7271% |
| Tercera, al cierre de la ventana | **98,7409%** |

No es un error de la herramienta: el denominador crece con el reloj, de modo que cada minuto sin incidentes mejora el indicador por sí solo. **Toda cifra de cumplimiento se emite sobre período cerrado**, y una lectura de período en curso es una foto de un contador en movimiento, no un resultado.

El corolario práctico es incómodo y conviene decirlo: **un indicador de período abierto siempre mejora con el tiempo si no ocurren incidentes nuevos.** Presentarlo como resultado permite elegir el momento de la captura para obtener el número deseado.

---

## 8. Recomendaciones

Ordenadas por impacto sobre la fiabilidad del indicador.

| # | Recomendación | Fundamento | Impacto |
|---|---|---|---|
| **1** | **Incorporar la revisión del modelo de servicio al procedimiento de alta de infraestructura** | El modelo se definió el 2026-08-07 y el borde de red se incorporó el 2026-08-11 sin revisarlo. Corregir solo este caso deja intacto el mecanismo que lo produjo | **Alto.** Control de proceso: evita que la brecha se reabra con la próxima incorporación |
| **2** | **Incorporar `austral-net` al árbol como dependencia de `Procesamiento de Pagos`** | Un componente cuya caída interrumpe el servicio no puede quedar fuera del indicador que declara cumplimiento | **Alto.** En el período declarado habría convertido un 100% en un 76,84% |
| **3** | **Definir cobertura de turno o política explícita de no cobertura fuera de horario** | El único incidente espontáneo del período no fue reconocido durante 74 minutos | **Alto.** Es la diferencia entre 3 y 74 minutos de interrupción |
| **4** | **Renombrar `Endpoint austral-ws01`** a una denominación que refleje su alcance real (p. ej. "Controles de seguridad — estación de operaciones") | El nombre actual induce a leer disponibilidad donde se mide cobertura de seguridad | Medio. Elimina un falso verde de lectura sin cambiar el cálculo |
| **5** | **Revisar la profundidad efectiva del escalamiento** | Un incidente de severidad alta sin reconocer durante 74 minutos generó dos notificaciones y ninguna posterior | Medio. Pendiente de verificación de configuración |
| **6** | **Unificar la zona horaria de las fuentes o declararla por fuente en todo timeline** | El registro del servidor opera en UTC y la consola en hora local: 4 horas de diferencia | Medio. Riesgo de error de causalidad en investigación de incidentes |
| **7** | **Emitir cumplimiento únicamente sobre período cerrado** | Las lecturas de período abierto son inestables por construcción y mejoran solas con el tiempo | Bajo. Disciplina de reporte |

---

## 9. Nota metodológica: exclusión declarada frente a exclusión configurada

La herramienta de monitoreo permite retirar tramos de tiempo del cálculo del indicador desde la configuración del objeto SLA. Se evaluó ese mecanismo y **se descartó deliberadamente** para este informe, por tres razones verificadas:

1. **Su efecto no es predecible.** Excluir un tramo no perdona indisponibilidad: retira el tiempo del cálculo completo, numerador y denominador. Si el tramo excluido era proporcionalmente más sano que el resto, el indicador **empeora**. En la prueba realizada, una exclusión de siete días bajó el indicador de 84,81% a 81,22%. Con exclusión total, el resultado es "no medido".
2. **No deja rastro en el reporte.** Quien tenga permiso de edición sobre un objeto SLA puede alterar el cumplimiento declarado sin modificar un solo dato de monitoreo, y el documento resultante no muestra que se excluyó nada.
3. **Existe una alternativa auditable**: declarar el período y sus exclusiones en el propio informe, con rangos exactos y justificación.

**La exclusión de la ventana de arranque V1 en este informe sigue esa alternativa.** El criterio general que se adopta:

> Excluir tiempo de un cómputo de cumplimiento es legítimo cuando queda escrito en el reporte con su rango y su motivo. Es ilegítimo cuando queda dentro de la herramienta, donde el lector no puede verlo.

Como consecuencia de gobernanza: **el permiso de edición sobre objetos SLA es un control de integridad de reportes, no una función operativa.** Debe separarse del rol que opera el centro de monitoreo y quedar sujeto a auditoría.

---

## 10. Limitaciones del entorno de medición

Se declaran en el cuerpo del informe, no en anexo, porque condicionan la lectura de todas las cifras anteriores.

- **La plataforma de virtualización que aloja el dispositivo de red presenta reinicios espontáneos sin causa registrada.** Se observaron tres durante la sesión previa y uno durante el período declarado. Es un riesgo de plataforma del entorno de laboratorio, no un incidente de infraestructura, y constituye la principal fuente de indisponibilidad medida en este informe.
- **Dos de los seis incidentes del período fueron provocados por el propio operador** para validar la instrumentación. Se identifican como tales en el registro y se reportan por separado en el agregado.
- **El único incidente espontáneo del período fue atendido por el mismo operador que redacta este informe**, sin turno formal. Los tiempos de respuesta demuestran que el ciclo es medible desde el historial; no constituyen métricas de una operación con cobertura definida.
- **No existe un acuerdo de nivel de servicio contratado.** Los SLO utilizados fueron definidos internamente por criterio de impacto. En una organización real derivan del contrato con el cliente, y el objetivo interno se fija por encima del comprometido para conservar margen de reacción.
- **Las caídas de enlace se provocan por desactivación administrativa de la interfaz**, no por interrupción de medio físico.
- **El período declarado abarca 5h 19m.** Es una ventana corta para evaluar cumplimiento y no permite afirmar tendencia. Su propósito es disponer de una medición interpretable, no representativa de un mes de operación.

---

## Anexo A — Método de verificación de la ventana declarada

La validez de todo el informe descansa en una afirmación: durante V2 el sistema de monitoreo estuvo operativo de forma continua. El método empleado para sostenerla:

**1. Continuidad del dato de sondeo.** Se verificó que el ítem de sondeo ICMP, con intervalo de 1 minuto, presentara serie continua durante toda la ventana. Los tramos con valor cero corresponden a indisponibilidad del objetivo; la ausencia total de muestras habría indicado indisponibilidad del monitoreo. **La distinción entre "el objetivo respondió que no" y "no hubo pregunta" es la que separa una medición de un hueco.**

**2. Correspondencia con el registro del servidor.** Cada transición de estado registrada en la consola se contrastó con su evento correspondiente en el registro del servidor de monitoreo —suspensión y rehabilitación del sondeo por indisponibilidad de interfaz—. La correspondencia fue exacta en los seis puntos de transición de la ventana.

Ambas fuentes son independientes: una proviene del almacenamiento de series temporales, la otra del proceso de recolección. Su coincidencia hace improbable un apagón no detectado.

**3. Conversión de zona horaria.** El registro del servidor opera en UTC y la consola en hora local (UTC−4). Toda correspondencia se estableció aplicando esa conversión de forma explícita.

> Esta última precaución no es un detalle de formato. **Un timeline de incidente que mezcla horas de registro con horas de consola sin declarar la conversión produce causalidades falsas**, y el error no es visible: ambas columnas parecen igualmente creíbles. La disciplina correspondiente es declarar la zona horaria de cada fuente en la primera columna de todo timeline de investigación.

---

## Anexo B — Registro de acciones posteriores a la emisión

> Este anexo se agrega el 2026-08-17. **El cuerpo del informe no se modifica**: un reporte de cumplimiento emitido no se reescribe, se anexa. Las cifras de las secciones 3 a 7 corresponden al estado del modelo de servicio al momento de la emisión.

| Recomendación | Estado | Fecha | Detalle |
|---|---|---|---|
| 1 — Revisión del modelo en el alta de infraestructura | **No implementada** | — | Control de proceso; queda como recomendación de gobernanza |
| 2 — Incorporar `austral-net` al árbol | **Implementada** | 2026-08-17 | Servicio `Conectividad de red (austral-net)` creado como hijo de `Procesamiento de Pagos`, alimentado por etiquetado selectivo. Validado de extremo a extremo con caída real |
| 3 — Cobertura de turno | **No implementada** | — | Fuera del alcance de un laboratorio operado por una persona. Se declara como no cubierto, no como resuelto |
| 4 — Renombrar `Endpoint austral-ws01` | **Implementada** | 2026-08-17 | Renombrado a `estacion de operaciones`, conservando el identificador interno para preservar historial y SLI. **Residual:** el SLA report lista los servicios en plano y ahí la ambigüedad subsiste |
| 5 — Revisar profundidad del escalamiento | **Verificada** | 2026-08-17 | La escalada termina en el minuto 10. No existe paso recurrente. La corrección —agregar un paso de rango abierto— queda como recomendación, no implementada |
| 6 — Zona horaria por fuente | **No implementada** | — | Mitigada declarando `UTC` en la plantilla de notificación |
| 7 — Cumplimiento sobre período cerrado | **Adoptada como disciplina** | — | Aplicada en este mismo informe |

**Verificación de integridad del reporte publicado.** Antes y después de modificar la estructura del árbol de servicios se leyeron los SLI de la semana cerrada 2026-08-09 / 08-15. Los tres valores coincidieron **a cuatro decimales** (84,8137 · 84,8134 · 98,8443). La corrección del modelo **no reescribió el pasado**: el cálculo se apoya en los eventos de estado ya registrados de cada servicio.

El servicio creado el 2026-08-17 figura como **`N/A`** en las semanas anteriores a su existencia, no como 100%.

**Nota de denominación.** La sección 3.2 de este informe nombra el servicio de seguridad con su denominación anterior, `Endpoint austral-ws01`. El renombramiento posterior aplica la recomendación 4 de este mismo documento y queda registrado acá para que la trazabilidad entre ambas denominaciones sea explícita.

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
