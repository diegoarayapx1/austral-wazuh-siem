# Bloque 4 — Operación NOC, capa de red y corrección del modelo de servicio

> AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.

**Sesiones:** 4A · 4B · 4C · 4D · 4E · 4F — del 11 al 17 de agosto de 2026
**Índice de evidencia:** [`../../evidence/README.md`](../../evidence/README.md)

Este documento reúne el bloque completo. La primera sección es la síntesis del bloque; las siguientes son el informe de cada sesión, en orden. La sesión 4D no produjo informe propio: su contenido —el incidente espontáneo y el reporte de disponibilidad— está en la síntesis y en `reports/`. La 4F fue empaquetado y cierre.

---

## Síntesis del bloque

### Capa de red, operación NOC y entregables de cierre

> **AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.** Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.

**Sesiones:** 4A · 4B · 4C · 4D · 4E · 4F (2026-08-11 a 2026-08-17)
**Duración acumulada:** ~15 h
**Entorno:** Zabbix 7.0.29 LTS sobre Docker Compose · dispositivo de red Cisco IOS (vIOS 15.9(3)M6) emulado en EVE-NG

> *Actualizado al cierre del proyecto (2026-08-17): incorpora los resultados de la sesión 4E y el empaquetado de la 4F.*

---

### 1. Alcance del bloque

Completar la capa de red del NOC, construir la superficie de operación diaria, y producir los entregables escritos del proyecto: el procedimiento de respuesta y el reporte de cumplimiento.

| Sesión | Contenido | Resultado |
|---|---|---|
| **4A** | Alta del dispositivo de red, SNMPv3, integración, caída de enlace | Capa de red incorporada al monitoreo |
| **4B** | Diagnóstico de fallo de notificación, muro NOC, ciclo alerta → ack → resolución | Muro operativo + primeras métricas de ciclo |
| **4C** | Validaciones, credenciales en macros protegidas, experimento de SLI, runbook | 4 deudas cerradas + runbook |
| **4D** | Reporte de disponibilidad, informe consolidado | Entregables de cierre |
| **4E** | Capa de red en el árbol de servicios, denominación, verificación del escalamiento | 3 deudas cerradas + 2 nuevas |
| **4F** | Empaquetado del repositorio, sanitización, acta de cierre | Proyecto cerrado |

### 2. Entorno

| Componente | Rol |
|---|---|
| `austral-noc` | Servidor Zabbix 7.0.29 (Docker Compose: server, web, base de datos) |
| `austral-net` | Router Cisco IOS emulado — borde de red, monitoreado por SNMPv3 authPriv (SHA + AES128) |
| `austral-app` / `austral-db` | Servidores Linux con agente |
| `austral-ws01` | Estación Windows con agente |

**Grupos de host:** `AustralPay/Red`, `AustralPay Servers`

---

### 3. Qué se hizo

#### 3.1 Capa de red (4A)

- Dispositivo `austral-net` desplegado en el emulador e integrado a la red del laboratorio.
- SNMPv3 authPriv con SHA + AES128 configurado en el equipo y en el host de monitoreo.
- Plantilla de fabricante vinculada; descubrimiento de interfaces con override del filtro de descubrimiento.
- Primera caída de enlace provocada por desactivación administrativa, con alerta verificada.

#### 3.2 Diagnóstico y superficie de operación (4B)

- Fallo de notificación de operaciones de actualización diagnosticado con nivel de registro elevado sobre los tres procesos relevantes del servidor. **Siete hipótesis refutadas con evidencia directa en base de datos.** Cerrado como declarado, con alcance acotado a una transición específica no reproducible.
- **Dependencia de supresión** declarada en el prototipo de trigger de enlace, apuntando al trigger de disponibilidad del dispositivo. Efecto en cascada: la rama completa de triggers de interfaz quedó suspendida bajo la disponibilidad del equipo. Una caída de dispositivo produce una alerta en lugar de hasta cinco por interfaz.
- **Muro NOC** con seis widgets, jerarquía de lectura definida y modo kiosko verificado.
- **Plantilla de notificación corregida**: eliminada una macro que resolvía a valor desconocido en triggers donde el campo no aplica; añadida ruta al runbook.
- **Ciclo completo** alerta → acknowledge → resolución ejecutado y capturado, con validación por contraste de que el acknowledge detiene la escalada.

#### 3.3 Validaciones, credenciales y procedimiento (4C)

- **Inventario de ítems cuadrado por enumeración directa**: 67 configurados = 63 con datos + 4 deshabilitados por hardware inexistente en la plataforma emulada, sin residuo.
- **Dependencia de supresión validada** tras rediseñar el test. El primer diseño no distinguía "la dependencia funcionó" de "el trigger nunca recibió el dato". Se provocó indisponibilidad ICMP manteniendo vivo el SNMP mediante filtrado selectivo, y con el estado operativo de la interfaz reportando caída con dato fresco, el trigger dependiente no generó evento.
- **Credenciales SNMPv3 migradas a macros de tipo secreto**, validadas contra el equipo por línea de comandos antes de guardarlas en un campo irrecuperable. El export del host queda limpio.
- **Experimento de reescritura retroactiva del indicador**, en dos brazos y cuatro intentos. La ventana de mantenimiento no altera el indicador hacia atrás; la exclusión de tiempo sí, con efecto no predecible.
- **Descripciones de trigger reescritas en origen**, con estructura verificación → ruta al runbook → criterio de escalamiento.
- **Runbook** con ocho triggers, matriz de escalamiento por impacto y anexo de lecturas engañosas de la consola.

#### 3.4 Reporte de cumplimiento (4D)

- Verificación de estado previa a cualquier cambio: contenedores, servicios, frescura de telemetría e integridad de la línea base del indicador.
- **Incidente espontáneo capturado**: caída del borde de red de 1h 14m, sin operador presente, con el sistema de monitoreo activo durante todo el tramo.
- Reconstrucción del incidente con **dos fuentes independientes** —historial de ítems y registro del servidor— con conversión explícita de zona horaria.
- **Hallazgo del modelo de servicio**: el borde de red no pertenece a ningún servicio del árbol; causa cronológica identificada.
- **Reporte de disponibilidad** sobre período declarado, con contraste calendario y contaminación documentada.

#### 3.5 Corrección del modelo de servicio (4E)

- **Capa de red incorporada al árbol** bajo `Procesamiento de Pagos`, mediante etiquetado selectivo en origen (`sla: network-core` en el trigger de alcance, `sla: network-{#IFNAME}` en el prototipo de enlace). Filtro del servicio con una sola condición por prefijo.
- **Experimento de control de retroactividad**: modificar la estructura del árbol **no** reescribe el SLI de períodos cerrados. El reporte publicado conserva su validez.
- **Validación con caída real** de extremo a extremo: hoja → rama → raíz, con `Root cause` propagado íntegro.
- **Padre duplicado corregido**: el servicio colgaba simultáneamente de la raíz y del servicio de negocio, porque el formulario precarga `Parent services` con el contexto de navegación.
- **Servicio de seguridad renombrado** conservando el identificador interno, de modo que historial y SLI se preservan.
- **Escalamiento verificado**: termina en el minuto 10, sin paso recurrente.

---

### 4. Hallazgos consolidados

Los hallazgos del bloque se agrupan en cuatro hilos. El valor de cada hilo está en su repetición: un patrón que aparece una vez es una anécdota; el mismo patrón en seis formas distintas es un principio operativo.

#### Hilo 1 — La consola no dice lo que uno cree que dice

| # | Manifestación | Lo que el operador concluiría mal |
|---|---|---|
| 1 | Host en verde con telemetría nula | "El equipo está bien" — la disponibilidad de interfaz no es la llegada de datos |
| 2 | Hueco de backoff indistinguible de host caído | "No hay datos porque el equipo está muerto" — puede ser que el monitoreo dejó de preguntar |
| 3 | Último chequeo con horas de antigüedad y el sistema funcionando | "El monitoreo está roto" — es el preprocesamiento de descarte de valores repetidos operando según diseño |
| 4 | Ausencia de dato y supresión por dependencia se ven idénticas | "La dependencia funcionó" — o el trigger nunca recibió el dato que lo dispara |
| 5 | Una interfaz nacida caída no genera alerta | "Falso negativo" — el trigger exige una transición, no un estado |
| 6 | Indicador en 100% con el borde de red caído 74 minutos | "El servicio estuvo disponible" — estuvo disponible lo que el modelo declara |

> **Formulación general:** toda pantalla de monitoreo es una interpretación, no una observación. La pregunta operativa no es "¿qué muestra?" sino "¿qué tendría que ser cierto para que muestre esto?". En cinco de los seis casos, la respuesta inequívoca estaba **una capa más abajo** — en el historial del ítem, en el registro del servidor, en la definición del servicio.

Este hilo es la razón por la que el runbook incluye un anexo de lecturas engañosas de la consola. Un procedimiento que asume que la pantalla dice la verdad falla exactamente en los casos difíciles.

#### Hilo 2 — Conclusión antes de verificar el instrumento

Ocho errores de método detectados y corregidos durante el bloque, todos con la misma estructura: una conclusión formulada sin comprobar antes el estado del instrumento o del tratamiento.

| # | Error | Qué lo delató |
|---|---|---|
| 1 | Consulta limitada presentada como conclusión sobre el universo | Un conteo sobre la tabla completa **cambió el diagnóstico entero** |
| 2 | Hipótesis de desfase de reloj sin verificar el reloj | Los timestamps coincidían al milisegundo |
| 3 | Silencio de un registro interpretado como dato, sin verificar que estaba activo | Al reelevar el nivel, la confirmación reportó que sí lo estaba |
| 4 | Experimento medido sobre período abierto | Un servicio no tratado subió exactamente lo mismo que el tratado |
| 5 | Tratamiento aplicado fuera de rango, no verificado | El período de la ventana apuntaba fuera del rango activo |
| 6 | Hipótesis formulada sobre una captura de otra sesión | El contraste directo contra el equipo la refutó en dos minutos |
| 7 | Estado del entorno declarado de memoria al cerrar sesión | La verificación de apertura de la sesión siguiente detectó la discrepancia en el primer minuto |
| 8 | Profundidad de escalamiento inferida de un contador de acciones | El contador registra acciones totales, no pasos de escalamiento |

> **Formulación general:** el error no es equivocarse de hipótesis — eso es el método funcionando. El error es **medir con un instrumento cuyo estado no se verificó**. Un registro que no se comprobó activo no produce evidencia negativa; un experimento sobre período abierto tiene una segunda variable moviéndose; un contador cuya semántica no se leyó no prueba lo que parece probar.

**Lo que salvó los experimentos, en las tres ocasiones:**

- **Grupo de control**, accidental o deliberado. Un efecto idéntico donde no hubo tratamiento no es efecto del tratamiento.
- **Bajar una capa.** El historial del ítem por sobre la lista de problemas; la base de datos por sobre el frontend; el registro del servidor por sobre la consola.
- **Verificar antes de un cambio irreversible.** Las credenciales se probaron contra el equipo antes de guardarlas en un campo que no se puede releer.

#### Hilo 3 — La medición hereda su significado del modelo, no de su nombre

| Caso | Mecanismo |
|---|---|
| El borde de red fuera del árbol de servicios | **Omisión.** El modelo se definió antes de que el dispositivo existiera en el inventario, y no se revisó al incorporarlo |
| Un servicio de endpoint reporta 100% con el endpoint apagado 16 minutos | **Ambigüedad.** Mide alcance de seguridad; el nombre sugiere disponibilidad |
| La exclusión de tiempo altera el cumplimiento sin dejar huella en el reporte | **Reescritura.** Retira tiempo del cálculo completo; puede subir, bajar o anular el indicador |
| El mismo dispositivo rinde 57,78%, 70,26% o 76,84% según la ventana | **Elección de período.** Ninguna cifra es falsa; el período elige el número |
| El indicador de período abierto dio tres valores distintos en una sesión | **Denominador móvil.** Mejora solo con el paso del tiempo si no hay incidentes nuevos |
| Un servicio creado hoy figura como `N/A` en períodos anteriores, no como 100% | **Negativa a medir.** La plataforma no emite cumplimiento sobre un período que no midió |

> **Formulación general:** un porcentaje de cumplimiento no es una medición, es el resultado de un modelo — qué componentes se incluyen, sobre qué ventana, con qué exclusiones. **El lector del reporte no ve el modelo: ve una etiqueta y un número.** Por eso todo indicador debe viajar con su período, su alcance y sus exclusiones declaradas, y por eso el permiso de edición sobre objetos de acuerdo de nivel de servicio es un control de integridad de reportes, no una función operativa.

Cerrado al término del 4E, el mapa completo de qué puede alterar un número ya publicado:

| Acción | ¿Reescribe el pasado? |
|---|---|
| Ventana de mantenimiento retroactiva | No |
| Modificar la estructura del árbol de servicios | No |
| `Excluded downtime` sobre el objeto SLA | **Sí** |

**Una sola palanca reescribe el cumplimiento, y es la que no deja rastro en el reporte.**

#### Hilo 4 — Lo automatizado falla en silencio

| Caso | Modo de fallo |
|---|---|
| Operaciones de actualización que no generan envío | Intermitente y no reproducible. Una ejecución exitosa histórica prueba que la configuración es correcta |
| Escalamiento sin paso final ante un incidente de 74 min sin reconocer | Sin error visible. Solo se nota contando notificaciones contra lo esperado |
| Mitigación en configuración volátil revertida por reinicio del equipo | Sin alerta. El cambio desaparece y nadie se entera |
| Campo de acción esperada con contenido inútil en la notificación | **Peor que no tener campo**: da falsa sensación de cobertura |
| Macro que resuelve a valor desconocido donde el campo no aplica | Sugiere dato faltante donde no corresponde el campo |
| El campo `Parent services` precargado con el contexto de navegación | El sistema actúa por el operador y no lo declara |

> **Formulación general:** una automatización rota **avisa**; una automatización degradada **no**. Todos los casos comparten que el sistema siguió funcionando y produciendo salida, con la salida equivocada. La contramedida no es más automatización: es contrastar periódicamente lo producido contra lo esperado. El registro de acciones, que documentó el ciclo aunque la notificación no saliera, es lo que mantuvo auditable la cadena de responsabilidad.

#### Hilo transversal — La sanitización por lectura de secciones no basta

Los secretos no se filtran por los bloques de configuración, donde todo el mundo los busca. Se filtran por **registros pegados, ejemplos de error y capturas de consola**, donde nadie los busca porque no parecen configuración.

El 4E amplió el hallazgo: el identificador del canal de notificación se propaga como **etiqueta automática generada por el propio sistema** en todo evento que originó notificación, visible en cualquier captura que expanda tags.

> La revisión de sanitización debe hacerse **por búsqueda de patrón sobre el documento completo**, y debe cubrir también el material gráfico — lo que exige conocer de antemano qué patrón buscar.

---

### 5. Métricas del bloque

#### 5.1 Ciclos de incidente medidos

| Ciclo | Naturaleza | MTTD | MTTA | MTTR (desde detección) | MTTR (desde falla real) |
|---|---|---|---|---|---|
| 2026-08-11 20:48 | Provocado, operador presente | — | 27 s | 60 s | — |
| 2026-08-17 11:54 | Provocado, operador presente | 2m 36s | 2m 37s | 13m 00s | 15m 36s |
| 2026-08-17 12:11 | Provocado (filtrado selectivo) | — | — | 8m 00s | — |
| **2026-08-17 13:55** | **Espontáneo, sin operador** | **2m – 3m** | **No aplica** | **1h 14m 00s** | **~1h 17m** |

**Progresión de credibilidad de la métrica.** Un MTTA de 27 s corresponde a un operador mirando la pantalla en el instante exacto de la detección. El de 2m 37s ya es una atención realista sobre una falla provocada. El del último ciclo no existe: **nadie tomó el incidente en 74 minutos**, y esa es la única cifra del bloque que refleja una condición operativa que no fue construida.

#### 5.2 Cumplimiento

**Período declarado — 2026-08-17, 11:40:33 → 17:00:00 (5h 19m 27s):**

| Servicio | SLO | SLI | Error budget consumido |
|---|---|---|---|
| Procesamiento de Pagos | 99,9% | 100% | 0% |
| Base de datos transaccional | 99,9% | 100% | 0% |
| Servidor de aplicación | 99,9% | 100% | 0% |
| Cobertura de controles de seguridad | 99% | 100% | 0% |
| **Borde de red (no modelado al momento del reporte)** | — | **70,26%** | — |

**Semana calendario cerrada 2026-08-09 / 08-15 — declarada no interpretable:**

| Servicio | SLI | Indisponibilidad implícita |
|---|---|---|
| Base de datos transaccional | 84,8137% | 25h 30m 46s |
| Procesamiento de Pagos | 84,8134% | 25h 30m 48s |
| Servidor de aplicación | 98,8443% | 1h 56m 29s |

> Los tres valores se releyeron en el 4E, tras modificar la estructura del árbol, y coincidieron **a cuatro decimales**. Es la prueba de que la corrección del modelo no reescribió el reporte publicado.

#### 5.3 Inventario del host de red

| Concepto | Cantidad |
|---|---|
| Ítems configurados | 67 |
| Ítems deshabilitados (hardware inexistente en la plataforma emulada) | 4 |
| Ítems con datos | 63 |

---

### 6. Entregables producidos

| Entregable | Ubicación |
|---|---|
| Host de red con SNMPv3 authPriv y credenciales en macros de tipo secreto | Configuración — export limpio |
| Dependencia de supresión de la rama de interfaz | Prototipo de trigger |
| Dashboard "muro NOC" | Configuración |
| Descripciones de trigger reescritas como procedimiento | Plantilla |
| Capa de red incorporada al árbol de servicios | Configuración |
| **Runbook de respuesta por trigger** | `runbooks/respuesta-triggers.md` |
| **Reporte de disponibilidad** | `reports/disponibilidad-2026-08-17.md` |
| Informe consolidado del bloque | `docs/bitacora/` |
| Estado del proyecto y deuda técnica | `docs/estado-y-deuda-tecnica.md` |

---

### 7. Evidencia

El índice completo, con la descripción de cada captura y la declaración de las no preservadas, está en **[`../../evidence/README.md`](../../evidence/README.md)**.

Se centraliza ahí a propósito: una tabla de evidencia duplicada en cada informe se desincroniza en cuanto se agrega, se descarta o se renombra un archivo. El índice es la fuente única.

Las capturas de este bloque están repartidas en `evidence/bloque4a/` a `evidence/bloque4e/`, una carpeta por sesión. La separación por carpeta resuelve la colisión de numeración entre sesiones sin alterar los nombres que citan los informes.

> **Correcciones aplicadas en la revisión de evidencia del 2026-08-17:**
> - La captura del incidente espontáneo se describía como *"problema activo de 1h 12m sin reconocer"*. Muestra el incidente **resuelto**, duración **1h 14m** — cifra que ya coincidía con la sección 5.1 de este informe. Se corrige la descripción; el MTTR no cambia porque ya era correcto.
> - La captura del árbol con la caída propagada estaba declarada como una sola. Son **tres**, una por nivel del árbol, y se publican como tal. Por separado ninguna prueba propagación.
> - Se incorpora la evidencia del renombramiento del servicio de seguridad (deuda #28), que no estaba en la tabla original.
> - Diecinueve capturas comprometidas en los informes no se preservaron. Están declaradas una por una en el índice.

---

### 8. Deuda técnica — estado al cierre del proyecto

| # | Ítem | Estado |
|---|---|---|
| 1 | Operaciones de actualización no generan envío | **Cerrado como declarado** — intermitente, 7 hipótesis refutadas, alcance acotado |
| 5 | Desfase de zona horaria entre frontend y registro del servidor | Documentado — recomendación emitida en el reporte |
| 6 | Descripciones de trigger en idioma distinto | **Cerrado** para los triggers de red |
| 7 | Disco de estación Windows sobre 90% | Aceptado |
| 8 | Host por defecto del servidor se auto-reactiva | Mitigado en el muro con grupos explícitos |
| 11 | Base de datos medida a nivel TCP | Declarado |
| 12 | Puerto de base de datos abierto a toda la red local | Pendiente |
| 13 | Endpoint de salud creado pero no consultado | Cosmético |
| 15 | Auto-registro sin clave precompartida | Deshabilitado tras la prueba |
| 16 | Problemas reconocidos escalan indefinidamente | Mitigado |
| 17 | Severidad uniforme en ambos enlaces | Identificado, no implementado |
| 18 | Descripciones de trigger explicaban la expresión lógica | **Cerrado** — reescritas en origen |
| 19 | Credenciales SNMPv3 en texto literal | **Cerrado** — macros de tipo secreto validadas |
| 20 | Interfaces ausentes del inventario | **Cerrado** — enumeración directa |
| 23 | Dependencia de supresión sin validar | **Cerrado** — validada con telemetría viva |
| 24 | Convención de nombres inconsistente en grupos de host | Identificado |
| 25 | Ruta al runbook duplicada en la notificación | Identificado |
| 26 | Editar la plantilla del fabricante se pierde ante una actualización | Declarado |
| **27** | Borde de red fuera del árbol de servicios | **Cerrada (4E)** — corregida y validada con caída real de extremo a extremo |
| **28** | Servicio de endpoint denominado como si midiera disponibilidad | **Cerrada (4E)** — renombrado; residual de lectura en vista plana declarado |
| **29** | Profundidad efectiva del escalamiento sin verificar | **Cerrada (4E)** — configuración leída; genera recomendación y deuda #31 |
| **30** | Servicio recién creado no evalúa un problema ya activo | **Nueva (4E)** — hipótesis de latencia de sincronización. Prueba: crear servicio, esperar 10 min sin tocar nada, provocar caída |
| **31** | Paso 3 del escalamiento sin notificación registrada en el incidente del 4D | **Nueva (4E)** — hipótesis: permiso del grupo de segundo nivel sobre el grupo de hosts. Prueba: `Users → User groups → Host permissions` |

**Balance del bloque:** ocho deudas cerradas más una acotada; siete deudas nuevas identificadas y declaradas.

**Recomendación derivada del hallazgo 6 del 4E, no implementada:** agregar un paso recurrente al escalamiento, con rango abierto, para que un problema no reconocido siga generando notificación indefinidamente.

---

### 9. Limitaciones declaradas del laboratorio

- **La plataforma de virtualización que aloja el dispositivo de red presenta reinicios y apagados espontáneos**, sin causa registrada. Patrón conocido de virtualización anidada. Es la principal fuente de contaminación de las métricas de disponibilidad y, a la vez, la fuente del único incidente espontáneo genuino del proyecto.
- **El indicador en período calendario no es interpretable** por el sesgo asimétrico del apagón del monitoreo: sobrecuenta lo ya abierto y subcuenta a cero todo lo demás. Los reportes se emiten sobre período declarado.
- **La dependencia de supresión se validó con una caída sintética**, construida para aislar la variable. La caída espontánea posterior **no la valida**: el trigger dependiente no recibió dato, de modo que su ausencia de evento no prueba supresión.
- **Tres de los cuatro ciclos de incidente fueron provocados por el mismo operador que los atendió.** Demuestran que el ciclo es medible desde el historial; no son métricas de una operación con cobertura definida.
- **Los tiempos de escalamiento del runbook se definieron por criterio de impacto propio**, no derivados de un acuerdo contratado.
- **Las caídas de enlace se provocan por desactivación administrativa**, no por corte de medio físico.
- **No existe acuerdo de nivel de servicio contratado.** Los objetivos de nivel de servicio fueron definidos internamente.

---

### 10. Estado del checklist del proyecto

- [x] Zabbix levantado + notificaciones funcionando
- [x] 3 hosts con agente reportando
- [x] SNMP al dispositivo de red operativo
- [x] Triggers por severidad configurados
- [x] Servicio + acuerdo de nivel de servicio + **reporte de disponibilidad**
- [x] Auto-discovery probado
- [x] Escalamiento de alertas validado
- [x] Ventana de mantenimiento configurada
- [x] Dashboard "muro NOC" armado
- [x] Caída simulada documentada (ciclo completo)
- [x] **Runbook redactado**
- [x] **README y empaquetado del repositorio**

**Proyecto 2 cerrado el 2026-08-17.**

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*

---

## Sesión 4A — Capa de red: SNMPv3 sobre equipo de borde emulado

**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Zabbix NOC · AustralPay Lab
Sesión: 4A

---

### Objetivo

Incorporar al NOC un dispositivo de red monitoreado por SNMPv3, cerrando la brecha entre el monitoreo de servidores —cubierto en los bloques anteriores— y la operación de red. El alcance abarca el despliegue del equipo en el emulador, la configuración de SNMPv3 con controles de seguridad en tres capas, la validación del protocolo con pruebas positivas y negativas, la integración en la plataforma de monitoreo y la producción de una caída de enlace detectada y notificada.

### Entorno

| Componente | Detalle |
|---|---|
| Plataforma de monitoreo | Zabbix 7.0.29 LTS sobre Docker Compose |
| Emulador de red | EVE-NG sobre virtualización anidada |
| Dispositivo de red | Router Cisco IOSv 15.9(3)M6 (`austral-net`) |
| Plantilla | `Cisco IOS by SNMP` |
| Protocolo | SNMPv3, nivel `authPriv`, SHA + AES128 |

**Topología del dispositivo:**

| Interfaz | ifIndex | Segmento | Propósito |
|---|---|---|---|
| `Gi0/0` | 1 | LAN de gestión | Alcanzable desde el servidor de monitoreo |
| `Gi0/1` | 2 | Segmento interno aislado | Enlace de prueba para simulación de caída |
| `Gi0/2`, `Gi0/3` | 3, 4 | — | Administrativamente caídas |

**Decisión de diseño — dos interfaces en vez de una.** Con una sola interfaz, la única forma de simular una caída sería apagar el equipo, y en ese escenario el monitoreo pierde también el SNMP: se observaría "host inalcanzable", no "interfaz caída". Con una segunda interfaz hacia un segmento aislado, la caída del enlace se produce **mientras el equipo sigue respondiendo y contando lo que pasó**.

### Estado inicial

El monitoreo de un dispositivo de red estaba planificado desde los primeros bloques y quedó sin ejecutar por una falla de la plataforma de emulación. Una revisión de alcance al abrir este bloque detectó que el componente había quedado sin asignación tras una reorganización previa entre bloques: figuraba como reubicado en un bloque cuyo informe no lo contiene.

Se declaró la brecha explícitamente y se evaluaron tres salidas: descopar el componente documentando la razón, sustituirlo por SNMPv3 contra un host Linux, o ejecutar la topología completa. Se optó por la tercera, con re-scope del bloque de dos a tres sesiones.

---

### Qué se hizo

#### Capa de plataforma

**1. Verificación de virtualización anidada antes de transferir la imagen.** El emulador ejecuta máquinas virtuales reales por cada nodo, de modo que la VM que lo aloja necesita recibir las extensiones de virtualización del anfitrión. Sin ellas la instalación se completa con normalidad pero los nodos se cuelgan en el arranque sin mensaje de error. La verificación válida se realiza desde dentro del sistema, no desde la interfaz de configuración del hipervisor.

**2. Despliegue de la imagen del router** respetando la convención de nombres del emulador: el prefijo del directorio determina la plantilla de hardware aplicada, y el archivo de disco debe adoptar un nombre fijo. Ninguna de las dos reglas produce error visible al violarse; el nodo simplemente no aparece o no arranca.

**3. Descarte deliberado de dos imágenes disponibles.** Un router de mayor porte se descartó por costo de recursos —tres a cuatro veces la memoria— sin aportar nada adicional para exponer la tabla de interfaces. Un switch de capa 2 se descartó porque no produce ningún dato adicional relevante para el objetivo.

#### Capa de configuración del dispositivo

**4. Configuración base** con direccionamiento en ambas interfaces y descripción explícita del propósito de cada una, verificada con alcance bidireccional contra el servidor de monitoreo.

**5. SNMPv3 con tres capas de control:**

| Capa | Implementación | Qué previene |
|---|---|---|
| Vista | Ramas de OID explícitamente incluidas y excluidas | Que la credencial de monitoreo enumere lo que no necesita |
| Lista de control de acceso | Únicamente la dirección del servidor de monitoreo | Uso de credenciales robadas desde otro origen |
| Modo de acceso | Solo lectura | Reconfiguración del equipo por vía SNMP |

La vista incluye la rama estándar de gestión y la rama privada del fabricante —necesaria para las métricas de CPU y memoria de la plantilla— y **excluye explícitamente las ramas de usuarios y de control de acceso del propio subsistema SNMP**: el usuario de monitoreo no puede enumerar la configuración de seguridad que lo autentica.

La lista de control de acceso se aplica a nivel de grupo y no de usuario, que es como lo implementa la plataforma del equipo. Tiene una ventaja de operación: cualquier usuario futuro incorporado al grupo hereda la restricción sin intervención adicional.

**6. Persistencia de identificadores.** Se fijaron dos parámetros que el equipo regenera al arrancar y que el monitoreo asume estables: el índice de interfaces y el identificador de motor SNMP. La justificación de cada uno aparece en los hallazgos.

**7. Verificación de ausencia de acceso por versiones anteriores.** Se confirmó que no existe ninguna community configurada. Los grupos de fábrica que aparecen en la configuración no habilitan acceso mientras no exista una.

#### Capa de validación del protocolo

**8. Cuatro pruebas ejecutadas por línea de comandos antes de tocar la plataforma de monitoreo**, dos positivas y dos negativas:

| Prueba | Resultado |
|---|---|
| Identidad del equipo | Descripción completa del sistema |
| Tabla de interfaces y estado operativo | Cinco interfaces con sus índices y estados |
| **Credencial incorrecta** | Fallo de autenticación explícito |
| **Consulta desde host no autorizado** | **Tiempo de espera agotado** |

**La diferencia entre las dos pruebas negativas es de diseño y no accidental.** La lista de control de acceso descarta el paquete antes de evaluar credenciales, de modo que el equipo no revela siquiera que existe un servicio SNMP escuchando. Credenciales válidas desde el origen equivocado no producen respuesta alguna.

La prueba se confirmó desde ambos extremos: tiempo de espera agotado en el cliente, y en el equipo el incremento del contador de denegaciones junto con la entrada de registro que identifica el origen rechazado. Un control verificado en un solo extremo no está verificado.

#### Capa de integración

**9. Alta del dispositivo en un grupo de host propio**, separado de los servidores: el runbook y el tablero de operación tratan red y cómputo como capas distintas.

**10. Host creado sin plantilla y validado con un único item de prueba** antes de integrar nada. La función de prueba de item consulta el equipo en vivo sin guardar configuración, sin plantilla, sin descubrimiento y sin caché intermedia. Es el mismo método de validación por capa aplicado en bloques anteriores al canal de notificación: **probar el componente aislado antes de integrarlo**.

**11. Plantilla enlazada** una vez confirmado el transporte, con 49 items, 20 triggers y 8 reglas de descubrimiento.

La plantilla emplea el patrón de item maestro con items dependientes: una única consulta recorre la tabla de interfaces completa y una regla de descubrimiento parsea ese bloque para crear los items individuales, que extraen su valor sin generar tráfico adicional. El motivo es de escala —un equipo de 48 puertos con ocho métricas por puerto serían cientos de consultas por ciclo frente a una— y tiene una consecuencia práctica: hasta que el item maestro recolecta y el descubrimiento se ejecuta, no existe ningún item de interfaz.

**12. Depuración del inventario descubierto.** Los items maestros correspondientes a hardware inexistente en un equipo emulado —ventiladores, fuentes de poder, sensores de temperatura— se deshabilitaron. Un item permanentemente no soportado ensucia el inventario sin generar ninguna señal que lo delate, según se documenta en los hallazgos. La verificación se hizo sobre la lista real de items en estado no soportado y no sobre una lista anticipada: contra lo previsto, el equipo emulado **sí** expone número de serie.

#### Escenario de caída

**13. Corrección del filtro de descubrimiento** mediante macro sobrescrita a nivel de host, dejando la plantilla intacta. La justificación completa aparece en el hallazgo central.

**14. Caída de enlace provocada** sobre la interfaz del segmento aislado, con el filtro ya corregido. Resultado: transición de estado operativo detectada, problema visible en la consola de operación con severidad media, y notificación entregada con identificador de incidente.

El trigger **permaneció habilitado durante todo el evento**, que es precisamente lo que la corrección buscaba demostrar.

---

### Hallazgos

#### Host en verde con telemetría nula

Durante el periodo de diagnóstico, la plataforma mostró el indicador de disponibilidad SNMP en verde y el item de disponibilidad del agente reportando estado disponible, mientras la totalidad de los items fallaba y no ingresaba ningún dato.

La causa es un criterio de diseño: los errores de configuración se tratan de forma distinta a los de red. Una sesión que no puede abrirse no cuenta como equipo inalcanzable, de modo que la disponibilidad de la interfaz nunca se degrada.

**Un operador que valida por semáforo observa un host disponible con telemetría nula.** El indicador en verde no significa que ingresan datos: significa que nada declaró lo contrario.

#### Consola de problemas vacía con la totalidad de los items en fallo

Complemento del anterior. Un item que no soporta su configuración no produce valor, y sin valor no hay expresión de trigger que evaluar. **La pantalla donde mira el operador permanece limpia mientras el host no entrega un solo dato.**

#### Una macro inexistente y una macro vacía son indistinguibles

El diagnóstico con nivel de registro elevado mostró que el servidor resolvía las macros de credenciales a cadena vacía. Ese síntoma corresponde a dos causas distintas: macro existente con valor vacío, y macro que no existe.

La causa real fue la segunda: las macros habían sido creadas con las **etiquetas de los campos de la interfaz** como nombre, en lugar de nombres de macro válidos, mientras la interfaz referenciaba identificadores que nunca existieron.

**La plataforma no advierte al guardar una interfaz que referencia una macro inexistente.** La acepta, la resuelve a cadena vacía en tiempo de sondeo, y el error final informa que la contraseña es demasiado corta. Tres capas de distancia entre la causa y el mensaje.

#### El cambio de tipo de macro borra el valor

Convertir una macro de tipo secreto a tipo texto elimina su valor por diseño, para impedir que un secreto se degrade a texto plano de forma accidental. Guardar sin reescribirlo deja la macro existiendo con contenido nulo, y el error resultante vuelve a acusar a la longitud de la contraseña.

En este caso **el procedimiento de diagnóstico introdujo su propio fallo**, desplazando el diagnóstico hacia la variable equivocada.

#### El descubrimiento de bajo nivel desactiva su propia alarma ante una caída administrativa

Al desactivar administrativamente la interfaz de prueba, el trigger de caída de enlace quedó marcado como no descubierto, programado para deshabilitarse en el siguiente ciclo y eliminarse en siete días.

La interfaz dejó de cumplir el filtro por defecto de la regla de descubrimiento, que excluye las interfaces administrativamente caídas. La plataforma la trató como recurso perdido y desactivó su trigger.

**El mecanismo de monitoreo interpretó la caída como desaparición del objeto monitoreado y desactivó la alarma destinada a detectarla.**

El argumento de que una desactivación administrativa es intencional no resuelve el problema: **el filtro no distingue quién la ejecutó.** Un administrador en ventana de mantenimiento y un actor con acceso al equipo producen el mismo resultado — la interfaz sale del inventario, el trigger se desactiva, y transcurrido el plazo desaparece la evidencia de que ese enlace existió. Es un mecanismo de evasión provisto por el propio sistema de monitoreo, cercano a la técnica de deterioro de defensas del marco MITRE ATT&CK, con la particularidad de que no requiere tocar la plataforma de monitoreo: basta con actuar sobre el equipo monitoreado.

**Remediado y verificado** mediante macro sobrescrita a nivel de host que neutraliza la exclusión, dejando la plantilla intacta. Tras la corrección, la misma acción produjo problema en consola y notificación entregada, con el trigger permaneciendo habilitado.

La contrapartida se declara: el inventario pasa a incluir interfaces apagadas de forma permanente. En un equipo de borde con dos interfaces relevantes el costo es nulo; en un equipo de acceso con decenas de puertos libres, el criterio correcto sería el inverso. **El filtro de descubrimiento es una decisión por tipo de equipo, no un valor global.**

#### Deshabilitar un trigger no lo pausa: destruye el evento activo

Cuando el descubrimiento deshabilitó el trigger, el problema desapareció de la consola y se emitió notificación de escalamiento cancelado. Al rehabilitarlo, **el problema no reapareció** pese a que la condición física seguía presente: la expresión exige una transición de valor, y el item reportaba el estado de fallo de forma estable.

La consecuencia operativa es significativa: si un operador deshabilita un trigger ruidoso durante un incidente y lo rehabilita después, **el incidente desaparece de la consola aunque el fallo continúe**. No se emite alerta nueva y el problema permanece invisible hasta el siguiente cambio de estado, que en un enlace caído de forma estable puede no producirse nunca.

Es la contrapartida exacta de un hallazgo del bloque anterior, donde deshabilitar un trigger fue la solución correcta para un item huérfano. El mismo mecanismo produce aquí un vacío de visibilidad. **La ventana de mantenimiento existe precisamente para este caso:** suprime la notificación sin destruir el evento.

#### El identificador de motor SNMP forma parte de la credencial

Tras un reinicio del equipo —necesario porque el emulador no permite modificar el cableado en caliente— el monitoreo volvió a fallar por error de autenticación, con las credenciales sin cambios y la configuración del usuario mostrándose correcta.

SNMPv3 no transmite la contraseña: ambos extremos derivan una clave localizada combinando la contraseña con el identificador de motor del equipo. Al regenerarse ese identificador en el arranque, las claves dejaron de coincidir.

El impacto operativo es concreto: tras un corte de energía en un sitio remoto, múltiples equipos dejan de reportar con error de autenticación, y un operador que confía en el mensaje dedica horas a revisar credenciales correctas.

**Remediado** fijando el identificador y recreando el usuario, en ese orden — fijar el identificador elimina todos los usuarios de versión 3, porque sus claves derivan de él.

Junto con la persistencia del índice de interfaces, son dos parámetros de la misma categoría: **estado que el equipo regenera al arrancar y que el monitoreo asume estable.** Sin persistencia del índice, una interfaz puede cambiar de posición tras un reinicio y todos los items quedan apuntando a la interfaz equivocada, sin producir ningún error.

El propio monitoreo capturó el incidente como problema de recolección SNMP, con apertura y cierre registrados.

#### La descripción de la interfaz se propaga hasta la consola

Los items descubiertos incorporaron como etiqueta el texto del campo de descripción configurado en el equipo. **La documentación escrita en la configuración del dispositivo llegó por sí sola hasta la consola de monitoreo**, lo que constituye un argumento verificable de por qué se describen las interfaces, más allá de la buena práctica declarativa.

#### El editor de topología no refleja el plano de datos con el nodo en ejecución

Eliminar el enlace en el editor no produjo caída de la interfaz: el equipo continuó reportando enlace activo con tráfico. En hardware físico, retirar el cable corta la portadora; en el emulador, el estado de enlace depende del puente subyacente y no del enlace dibujado, y la topología no se reaplica sin detener el nodo.

Limitación del entorno de emulación, relevante al diseñar escenarios de caída.

---

### Nota metodológica

El diagnóstico del fallo de sesión SNMP recorrió tres hipótesis, dos de las cuales resultaron refutadas por la evidencia posterior. La causa real apareció en la primera lectura del registro con nivel de depuración elevado.

Una de las hipótesis refutadas merece registro por el tipo de error que la sostuvo: una tabla de resultados internamente consistente apuntaba a una incompatibilidad de cifrado, y estaba equivocada porque **dos variables cambiaron simultáneamente** — el protocolo de cifrado y, sin advertirlo, el estado del usuario en el equipo, que había sido recreado varias veces durante la sesión. Una tabla de resultados consistente no constituye prueba si las condiciones no estaban controladas.

**La conclusión operativa: elevar el nivel de registro corresponde antes de formular hipótesis, no después de agotarlas.**

Se registran además tres fallos silenciosos de procedimiento observados durante la sesión: una prueba negativa ejecutada desde el origen autorizado, que resulta indistinguible de una prueba exitosa y no valida nada; un patrón de invocación de contenedores heredado de un proyecto previo que, al no coincidir el filtro, ejecuta el comando con un argumento desplazado y devuelve un error que apunta a otra causa; y la elevación del nivel de registro sin provocar un sondeo intermedio, que produce un registro vacío interpretable como ausencia de error.

---

### Limitaciones declaradas

- **El dispositivo de red es emulado, no físico.** No expone sensores ambientales, y su estado de enlace no responde al cableado del editor con el nodo en ejecución.
- **La caída se provoca por desactivación administrativa**, no por corte de medio físico. Un corte real produce estado operativo caído con estado administrativo activo; el escenario ejecutado produce ambos caídos. La corrección del filtro de descubrimiento cubre esa diferencia, pero el escenario no es idéntico a una falla de medio.
- **Credenciales de laboratorio en texto literal** en la configuración del host. En producción corresponderían macros de tipo secreto o un gestor externo de secretos. Se declara la distinción relevante: una macro secreta no se muestra en la interfaz, no viaja en los exports ni aparece en la API, pero **su valor permanece sin cifrar en la base de datos**. La protección es contra exposición accidental, no contra acceso directo al almacenamiento.
- **Identificador de motor SNMP fijado como remediación**, no como política de despliegue inicial.
- **Un solo dispositivo de red.** El valor del descubrimiento automático se aprecia con decenas de interfaces; con dos, items explícitos serían igualmente claros. La plantilla se emplea por realismo operativo, no por necesidad de escala.

---

### Estado al cierre

El dispositivo de red está monitoreado por SNMPv3 con autenticación y cifrado, vista restringida y control de acceso por origen, integrado en la plataforma con plantilla del fabricante, descubrimiento automático operativo y filtro corregido. La caída de enlace se encuentra detectada, visible en consola y notificada.

Pendientes declarados para las sesiones siguientes: reintroducción de las credenciales como macros secretas, diferenciación de severidad entre el enlace de gestión y el enlace de prueba mediante override en el prototipo de trigger, y reescritura de las descripciones de los triggers heredados —que actualmente explican la lógica de la expresión en lugar de indicar el procedimiento de respuesta, deficiencia que se aborda junto al runbook.

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*

---

## Sesión 4B — Muro de operación y ciclo de incidente

**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Zabbix NOC · AustralPay Lab
Sesión: 4B

---

### Objetivo

Tres objetivos consecutivos: diagnosticar el fallo de notificación de operaciones de actualización arrastrado desde el Bloque 2, construir el muro de operación del NOC incorporando la capa de red, y documentar un ciclo completo de incidente —detección, asunción de responsabilidad y resolución— con métricas obtenidas del historial real de la plataforma.

### Entorno

| Componente | Detalle |
|---|---|
| Plataforma de monitoreo | Zabbix 7.0.29 LTS sobre Docker Compose |
| Base de datos | PostgreSQL 16 |
| Dispositivo de red | Router Cisco IOSv monitoreado por SNMPv3 |
| Hosts monitoreados | Dos servidores Linux, un endpoint Windows, un dispositivo de red |
| Agrupación | Grupo dedicado de red · grupo de servidores |

### Estado inicial

Al abrir la sesión, el enlace de prueba que se había dejado deliberadamente caído al cierre del bloque anterior se encontraba operativo: el dispositivo se había reiniciado y la desactivación administrativa, aplicada solo a la configuración en ejecución, se revirtió. El escenario de prueba se reconstruyó desde cero.

---

### Qué se hizo

#### Capa de diagnóstico

**1. Instrumentación previa a la formulación de hipótesis.** Se elevó el nivel de registro únicamente de los tres procesos implicados en la cadena de notificación —el que decide si corresponde notificar, el que entrega, y el que consume las tareas encoladas por la interfaz—, en lugar del servidor completo. Elevar el nivel global habría sepultado lo relevante bajo cientos de líneas por minuto del sondeo SNMP.

**2. Diseño de un control del experimento.** Antes de cada prueba se verificó que la notificación del evento de problema llegara efectivamente por el canal, sobre ese mismo evento. Sin ese control, la ausencia de notificación tras una actualización manual es ambigua: no distingue un fallo específico de la funcionalidad de un fallo general de la cadena.

**3. Descarte de un caso de prueba inválido.** El único problema activo al iniciar era de severidad inferior al umbral configurado en la acción. Una actualización sobre ese evento legítimamente no debe notificar, de modo que diagnosticar sobre él habría producido una causa raíz falsa. Se generó un problema de severidad adecuada.

**4. Verificación de la cadena eslabón por eslabón, contra la base de datos y no contra la interfaz.** La interfaz muestra la configuración editada; la base de datos muestra la persistida, que es la que el servidor evalúa.

| Eslabón verificado | Resultado |
|---|---|
| Acción habilitada, con la fuente de evento correcta | Correcto |
| Operaciones de actualización persistidas en su fase | Correcto — dos operaciones registradas |
| Destinatario y canal poblados | Correcto |
| Ausencia de condiciones bloqueantes sobre la operación | Correcto — las únicas condiciones existentes corresponden a los pasos de escalamiento |
| Plantilla de mensaje específica para actualizaciones | Correcto — presente y personalizada |
| Registro de la actualización manual | Correcto |
| Tarea encolada por la interfaz y consumida por el servidor | Correcto — marcada como completada |
| **Generación de la escalada asociada a la actualización** | **Ausente** |
| **Generación del registro de alerta asociado** | **Ausente** |

Siete hipótesis formuladas y refutadas con evidencia directa.

**5. Hallazgo que reclasifica la falla.** Un recuento sobre la tabla completa —en lugar de una consulta acotada a las últimas filas— reveló una ejecución exitosa histórica de la funcionalidad, con la misma acción, el mismo canal, el mismo usuario y la misma severidad que los intentos fallidos. La configuración es por tanto correcta, y la falla se reclasifica de inoperante a **intermitente y no reproducible**.

**6. Última hipótesis falsable, probada y descartada.** El único elemento que diferenciaba la ejecución exitosa de las fallidas era el estado de la escalada del evento al momento de la actualización: agotada en el caso exitoso, activa o inexistente en los fallidos. Se probó dirigidamente sobre un evento con escalada agotada y no reprodujo.

**7. Cierre por proporcionalidad.** Excedido el plazo acordado y agotadas las hipótesis razonables, se cerró el ítem como declarado, con alcance acotado, y se restituyeron los niveles de registro.

#### Capa de reducción de ruido

**8. Dependencia de supresión en la capa de red.** Una caída del dispositivo durante la sesión produjo dos problemas simultáneos con una sola causa raíz: indisponibilidad del equipo y caída del enlace. El dispositivo se incorporó al monitoreo con posterioridad al bloque en que se declararon las dependencias del resto de los hosts, de modo que no las había heredado.

La dependencia se declaró **en el prototipo de trigger, no en el trigger generado**: el trigger generado se sobrescribe en el siguiente ciclo de descubrimiento automático.

**Efecto en cascada no previsto:** los restantes prototipos de interfaz de la plantilla del fabricante —degradación de velocidad, uso elevado de ancho de banda, tasa de errores— ya dependían del trigger de caída de enlace. Al colgar este de la disponibilidad del equipo, la rama completa quedó suspendida bajo ella. Un equipo caído genera una alerta en lugar de hasta cinco por interfaz.

#### Capa de visualización

**9. Muro de operación** con seis componentes y jerarquía de lectura explícita:

| Zona | Componente | Criterio |
|---|---|---|
| Superior izquierda | Lista de problemas activos, ordenada por severidad descendente | Lo que exige acción inmediata, con la mayor superficie |
| Superior derecha | Conteo por severidad y grupo de host | Estado de un vistazo, separando red de servidores |
| Media derecha | Hosts afectados por grupo | Alcance del impacto |
| Media izquierda | Disponibilidad por transporte de monitoreo | Salud de la propia recolección |
| Inferior | Dos informes de nivel de servicio | Tendencia y contexto |

Decisiones de diseño aplicadas:

- **Umbral de severidad alineado con el de la acción de notificación.** Mostrar severidades que nadie va a atender entrena al operador a ignorar regiones de la pantalla.
- **Sin componente dedicado de red.** El conteo por grupo ya provee la distinción; un componente adicional habría duplicado información y restado espacio. El muro se diseña por lo que se excluye.
- **Grupos sin problemas visibles.** Un grupo en cero es información: prueba que el muro observa ese grupo. Ocultarlo impide distinguir un grupo sano de uno no monitoreado.
- **Grupos de host declarados explícitamente** en el componente de hosts afectados, para excluir el host interno que la instalación crea por defecto y que se reactiva por sí solo.
- **Rango temporal de los informes de nivel de servicio acotado** al período con datos, eliminando columnas vacías.

**10. Modo kiosko verificado.** Se declara la implicación de seguridad: un muro en kiosko es una sesión autenticada permanentemente abierta. La configuración correcta exige un usuario dedicado de solo lectura, restringido a los grupos exhibidos, y aseguramiento físico del terminal. El muro es simultáneamente un control operativo y una superficie de exposición.

#### Capa de notificación

**11. Corrección de la plantilla de mensaje de problema**, en dos puntos:

- **Eliminación del campo de técnica de ataque**, que resolvía a un valor de marcador en toda alerta sin la etiqueta correspondiente. La ausencia de la etiqueta es correcta —una caída de enlace no es una técnica de ataque—, pero el mensaje sugería un dato faltante donde el campo simplemente no aplica. La información permanece disponible en la etiqueta del evento para los triggers que sí la llevan.
- **Incorporación de una ruta al runbook**, separada visualmente del campo de procedimiento.

Fundamento de la decisión. La plataforma no admite condicionales en el cuerpo del mensaje, de modo que una plantilla única no puede servir bien a dos clases de trigger con y sin procedimiento propio. Se evaluaron y descartaron dos alternativas: eliminar el campo de procedimiento habría producido un mensaje más limpio que no responde la única pregunta que hace accionable una alerta; y reescribir las descripciones heredadas exige, si el campo resulta no editable en objetos de plantilla, clonar la plantilla del fabricante y asumir su mantenimiento fuera del ciclo de actualizaciones. La solución adoptada **degrada correctamente**: donde existe procedimiento propio se lee el procedimiento; donde el texto heredado es inútil, el operador dispone de una ruta que sí lo es.

#### Ciclo de incidente

**12. Ciclo completo ejecutado y medido** sobre el enlace de prueba, con el muro montado:

| Fase | Registro |
|---|---|
| Detección y notificación | Problema en consola, notificación entregada |
| Asunción de responsabilidad | Acuse de recibo con mensaje de turno, visible en el registro de acciones |
| Resolución | Estado resuelto con tiempo de recuperación y duración calculada, notificación de recuperación entregada |

| Métrica | Valor |
|---|---|
| Tiempo medio de reconocimiento | 27 segundos |
| Tiempo medio de resolución | 60 segundos |

**13. Validación no planificada de la detención del escalamiento.** El contraste entre dos ciclos consecutivos lo demuestra: el primero, sin acuse de recibo oportuno, registra tres operaciones —los tres pasos de la cadena de escalamiento—; el segundo, con acuse a los 27 segundos, registra dos. El acuse de recibo cortó la cadena en el primer paso. Es la validación del bloque de escalamiento replicada sobre la capa de red, con evidencia de contraste.

---

### Hallazgos

#### La notificación de actualización es intermitente, no inoperante

Resultado principal de la sesión. Siete hipótesis de configuración refutadas con evidencia directa en base de datos, y una ejecución exitosa histórica que prueba que la configuración es correcta. La falla se ubica entre el procesamiento de la tarea de acuse de recibo y la generación de la escalada asociada, y no es reproducible.

La distinción importa operativamente: una funcionalidad inoperante se corrige con configuración; una intermitente exige instrumentación a nivel de proceso o reproducción en una instancia limpia, ambas fuera del alcance del laboratorio.

**Consecuencia concluyente:** el registro de acciones consigna siempre la actualización, con usuario, marca de tiempo y mensaje del operador. El ciclo de responsabilidad es auditable con independencia de la notificación.

#### Una mitigación aplicada solo en la configuración en ejecución desaparece sin dejar rastro

La desactivación administrativa del enlace, nunca escrita a la configuración de arranque, se revirtió con el reinicio del equipo, sin alerta ni registro. Es un patrón de incidente real: un cambio aplicado se pierde por un reinicio de causa independiente y el sistema vuelve a un estado que nadie está observando. La contracara es simétrica: persistir un cambio incorrecto lo vuelve permanente. La disciplina correspondiente es programar un reinicio de seguridad antes de intervenir un dispositivo remoto.

#### La descripción de un runbook que explica la expresión lógica es peor que su ausencia

El campo de procedimiento de las notificaciones de triggers heredados entregaba una explicación de la lógica booleana de la expresión, con macros de ejemplo sin resolver. La notificación funciona y el procedimiento no: el operador recibe la alerta correcta, en el canal correcto, con la instrucción equivocada. Esto produce falsa sensación de cobertura, que es un resultado peor que un campo vacío.

#### Un campo no aplicable no debe informarse como dato faltante

El campo de técnica de ataque resolvía a un valor de marcador en toda alerta de red. La ausencia de la etiqueta es una decisión correcta de clasificación, pero su presentación sugería una omisión. Distinción relevante al diseñar plantillas que sirven a dominios heterogéneos.

#### La suspensión automática de sondeo produce un hueco indistinguible de una falla de monitoreo

Ante fallos consecutivos, el servidor suspende los chequeos del host y reintenta periódicamente. Durante esa suspensión no se recolectan datos, y el vacío resultante no distingue un equipo caído de un monitoreo detenido. Es la misma clase de ceguera que una ventana de mantenimiento sin recolección, analizada en un bloque previo.

#### Dos alertas de distinta severidad para una sola causa raíz

Sin dependencia declarada, la caída de un equipo de red genera simultáneamente una alerta de indisponibilidad y una de caída de enlace. El operador debe deducir que el enlace no está caído: el equipo entero lo está. Corregido mediante dependencia en el prototipo.

#### La convención de nombres de grupos de host tiene efecto funcional

Los grupos del entorno emplean dos convenciones distintas, una con separador jerárquico y otra sin él. El separador no es cosmético: habilita el anidamiento de grupos y la propagación de permisos sobre subgrupos.

---

### Nota metodológica

La sesión aplicó desde el inicio la conclusión metodológica del bloque anterior —elevar el nivel de registro antes de formular hipótesis— y aun así incurrió en tres errores de razonamiento que se registran por ser reutilizables.

**Una consulta acotada no autoriza una conclusión sobre el universo.** Se afirmó la ausencia total de un tipo de registro a partir de las últimas diez filas de una tabla. Un recuento sobre la tabla completa devolvió un resultado distinto y **modificó el diagnóstico por completo**, de funcionalidad inoperante a intermitente. Es el error más costoso de la sesión y el más fácil de evitar.

**Una hipótesis no se propone antes de verificar el hecho que la sostiene.** Ante un comportamiento anómalo de la herramienta de consulta de registros, se propuso un desfase de reloj entre componentes. La verificación posterior mostró coincidencia al milisegundo. La anomalía queda documentada como comportamiento observado, sin causa confirmada.

**El esquema se verifica antes de escribir la consulta, no después del error.** Dos consultas fallaron por nombres de columna incorrectos, en ambos casos porque se asumió una estructura plausible en lugar de inspeccionar la real. La inspección cuesta segundos; cada error consumió una iteración del plazo acordado.

Se registran además dos fallos silenciosos del entorno: la nomenclatura interna de los procesos del servidor no coincide con la del código y debe leerse del propio registro, y la consulta de registros del contenedor con filtro temporal relativo devuelve vacío mientras el filtro por número de líneas opera con normalidad.

Finalmente, se confirma un principio del bloque anterior en versión más sutil: **un instrumento no verificado no produce evidencia negativa**. El silencio de un registro solo es un dato si se ha comprobado previamente que el registro está activo.

---

### Limitaciones declaradas

- **Los valores de nivel de servicio exhibidos en el muro son reales pero contaminados.** Reflejan un entorno de laboratorio que se apaga, se suspende y se reinicia, no una infraestructura degradada. Se declaran como tales y se analizan en el reporte de disponibilidad correspondiente.
- **El nodo emulado se apaga espontáneamente**, sin apagado ordenado ni causa registrada. Patrón conocido de virtualización anidada. Riesgo de plataforma, no incidente de infraestructura, y fuente adicional de contaminación de las métricas.
- **Las métricas de reconocimiento y resolución provienen de un ciclo único**, ejecutado por el mismo operador que provocó la falla. No constituyen métricas de operación: demuestran que el ciclo es medible desde el historial de la plataforma. En operación real, un reconocimiento en 27 segundos implicaría un operador observando la pantalla en el instante exacto de la detección.
- **La dependencia de supresión quedó configurada pero no validada** con una caída posterior a su aplicación.
- **El componente de disponibilidad informa hosts en estado desconocido** para un transporte de monitoreo que el entorno no utiliza por decisión de diseño. El componente informa desconocimiento donde no tiene datos, que es el comportamiento correcto.
- **La caída de enlace se provoca por desactivación administrativa**, no por corte de medio físico. Limitación heredada del bloque anterior.

---

### Estado al cierre

El fallo de notificación de actualizaciones queda documentado con alcance acotado: siete hipótesis de configuración refutadas con evidencia directa, una ejecución exitosa histórica que descarta la configuración como causa, y la falla localizada en una transición específica no reproducible. La auditabilidad del ciclo de responsabilidad queda cubierta por el registro de acciones.

El muro de operación está montado con jerarquía de lectura definida, capa de red incorporada y modo kiosko verificado, con sus implicaciones de seguridad declaradas. La rama completa de triggers de interfaz quedó suspendida bajo la disponibilidad del dispositivo. La plantilla de notificación entrega una ruta accionable en lugar de un campo no aplicable.

El ciclo de incidente está documentado end-to-end con métricas obtenidas del historial real y con validación por contraste de que el acuse de recibo detiene la cadena de escalamiento.

Pendientes declarados para la sesión de cierre: reescritura de las descripciones de los triggers heredados junto al runbook, reporte de disponibilidad con contaminación declarada, experimento de ventana de mantenimiento retroactiva, reintroducción de credenciales como macros secretas, y validación de la dependencia de supresión con una caída real.

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*

---

## Sesión 4C — Validaciones, secretos y runbook

**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Zabbix NOC · AustralPay Lab
Sesión: 4C

---

### Objetivo

Cerrar las validaciones pendientes de la sesión anterior, eliminar las credenciales en texto literal que impedían exportar la configuración del dispositivo de red al repositorio, determinar empíricamente si la plataforma permite reescribir el indicador de nivel de servicio de forma retroactiva, y producir el runbook de respuesta por trigger.

### Entorno

| Componente | Detalle |
|---|---|
| Plataforma de monitoreo | Zabbix 7.0.29 LTS sobre Docker Compose |
| Base de datos | PostgreSQL 16 |
| Dispositivo de red | Router Cisco IOSv monitoreado por SNMPv3, nivel `authPriv` |
| Hosts monitoreados | Dos servidores Linux, un endpoint Windows, un dispositivo de red |

### Estado inicial

Se ejecutó una verificación de estado previa a cualquier cambio, con el fin de distinguir problemas preexistentes de los causados por la sesión.

La verificación detectó una discrepancia con el informe de cierre de la sesión anterior: el enlace de prueba figuraba con un problema activo de más de cinco días, mientras el informe declaraba el entorno sin problemas de red. La causa se documenta en el hallazgo 1.

**Lección de método aplicable:** el estado del entorno al cierre de una sesión debe verificarse contra la consola, no declararse desde la memoria de trabajo.

---

### Qué se hizo

#### Capa de validación

**1. Verificación del inventario de items por enumeración.** La sesión anterior había dado el punto por resuelto a partir del crecimiento del conteo total, lo cual es inferencia. Se enumeraron las cuatro interfaces con sus nueve items cada una mediante el subfiltro de etiquetas.

El inventario cuadra sin residuo: 67 items configurados menos 4 deshabilitados —tres correspondientes a hardware que el dispositivo emulado no expone, uno duplicado— igualan los 63 con datos.

**2. Validación de la dependencia de supresión — primer intento, descartado.** Se detuvo el nodo completo en el emulador. El resultado observado fue un único problema en consola, lo que aparentaba validar la dependencia.

**No validaba nada.** El historial del item de estado operativo no registró ningún valor posterior a la detención: el último fue "activo". El dispositivo perdió gestión y plano de datos simultáneamente, de modo que el trigger dependiente nunca recibió el dato que lo dispara. El silencio observado era ausencia de insumo, no supresión.

Los dos casos —dependencia efectiva y ausencia de dato— **son indistinguibles en la vista de problemas**. Concluir desde ahí habría producido una validación falsa.

**3. Rediseño del experimento para aislar la variable.** El defecto del primer diseño era que la causa raíz eliminaba ambas señales a la vez. Se construyó una condición donde la indisponibilidad de alcance coexiste con telemetría viva: una lista de control de acceso en la interfaz de gestión que descarta únicamente las pruebas de alcance procedentes del servidor de monitoreo. Con el trigger maestro ya en estado de problema, se desactivó administrativamente el enlace de prueba.

| Señal | Estado |
|---|---|
| Trigger maestro de indisponibilidad | En problema |
| Estado operativo del enlace | Reportado como caído, con dato fresco |
| Trigger de caída de enlace | Ningún evento generado |
| Total de problemas del dispositivo | Uno |

El trigger dispuso del dato que lo dispara y no generó evento. La dependencia queda **validada sin ambigüedad**: una caída de dispositivo produce una alerta en lugar de hasta cinco por interfaz.

#### Capa de gestión de secretos

**4. Verificación previa al cambio irreversible.** Las credenciales se validaron contra el dispositivo por línea de comandos **antes** de almacenarlas en campos de tipo secreto, que no admiten lectura posterior. Un error de transcripción descubierto después habría obligado a rehacer la configuración sin posibilidad de diagnosticar cuál de los tres valores falló.

La validación se ejecutó como contraste de una sola variable —protocolo de cifrado— con resultados opuestos: el algoritmo correcto devolvió el valor esperado y el incorrecto devolvió tiempo de espera agotado. Ese contraste refutó además una hipótesis de trabajo sobre un supuesto cambio de depuración que habría quedado en producción.

**Detalle de diseño del protocolo:** un cliente con el algoritmo de cifrado equivocado no obtiene error de autenticación sino ausencia de respuesta. El dispositivo no confirma siquiera que el usuario exista.

**5. Migración de credenciales a macros de tipo secreto.** Las tres credenciales —identificador de usuario y ambas frases de paso— se trasladaron a macros a nivel de host, y los campos de la interfaz pasaron a referenciarlas. Validado con telemetría fresca posterior al cambio.

**Alcance real del control, declarado:** este tipo de macro no cifra. El valor se almacena en la base de datos de la plataforma y es legible por quien tenga acceso a ella. Lo que resuelve es la exposición en la **interfaz**, en la **interfaz de programación** y en el **archivo de exportación**. Es un control de superficie, no de confidencialidad.

La respuesta a la confidencialidad es un gestor externo de secretos, donde la macro almacena una ruta y el valor nunca toca la base de datos de la plataforma. En un entorno con relevancia PCI-DSS, el control aplicado aquí no sería suficiente.

#### Capa de medición de servicio

**6. Experimento sobre reescritura retroactiva del indicador de nivel de servicio.** Se diseñó en dos brazos, porque la plataforma ofrece dos mecanismos independientes que suelen confundirse: la ventana de mantenimiento, que suprime problemas y notificaciones en tiempo real, y la exclusión de período, que retira tiempo del cálculo del indicador.

La predicción se registró antes de ejecutar: el primer mecanismo no alteraría el indicador retroactivamente, el segundo sí.

Se requirieron cuatro intentos. Los tres primeros presentaron defectos de diseño, todos detectados antes de formular conclusión:

| Intento | Defecto | Lección |
|---|---|---|
| 1 | Medición sobre período abierto: el denominador crecía con el reloj | Un experimento sobre nivel de servicio exige período cerrado |
| 2 | El período de la ventana caía fuera de su rango de vigencia, de modo que nunca se aplicó | Verificar que el tratamiento se aplicó efectivamente |
| 3 | Exclusión de una hora sobre un tramo sin indisponibilidad conocida | El tratamiento debe cubrir indisponibilidad conocida |
| 4 | — | Resultado válido |

El primer intento se detectó gracias a un control accidental: **un servicio no tratado varió exactamente lo mismo que el tratado.** Un efecto idéntico donde no hubo tratamiento no es efecto del tratamiento. El artefacto del reloj se observó de forma consistente en cuatro mediciones sucesivas sin tratamiento activo.

**Resultado sobre período cerrado:** la ventana de mantenimiento retroactiva no modificó el indicador en ninguno de los tres servicios. La exclusión de período sí lo modificó, y además lo eliminó por completo cuando cubrió la totalidad del período.

#### Capa de procedimiento

**7. Corrección en origen de las descripciones de trigger.** Se verificó que el campo de descripción es editable en los prototipos heredados de plantilla de fabricante, lo que permite corregir el defecto en origen en lugar de mitigarlo.

Las descripciones heredadas explicaban la expresión lógica del trigger. Dado que la plantilla de notificación las inserta bajo un encabezado de acción esperada, el operador recibía álgebra booleana donde debía haber una instrucción.

Se reescribieron las tres descripciones de los triggers que un operador de primer nivel recibe efectivamente, con estructura de verificación, ruta al procedimiento y umbral de escalamiento.

**Costo declarado:** editar la plantilla del fabricante implica que una actualización futura de esa plantilla sobrescribe el cambio. Aceptable con un dispositivo; en producción la práctica correcta es una plantilla propia enlazada además de la del fabricante.

**8. Runbook de respuesta por trigger.** Ocho triggers cubriendo capa de red y de servidores, cada uno con cinco secciones: qué significa, impacto, verificación, acción y escalamiento. Incluye matriz de escalamiento con tiempos derivados del impacto sobre el servicio —no de la severidad del trigger— y un anexo de lecturas engañosas de la consola.

El contenido de verificación procede en su mayor parte de fallas producidas y diagnosticadas en este laboratorio.

**9. Validación de extremo a extremo.** Se provocó una caída y la notificación entregada contenía el criterio de verificación completo, la ruta al procedimiento y el umbral de escalamiento.

---

### Hallazgos

#### 1. Asimetría del sesgo por interrupción del monitoreo

El enlace de prueba registró un problema activo durante **cinco días y catorce horas**, período en el cual el laboratorio completo —incluido el servidor de monitoreo— estuvo apagado. La interfaz no estuvo caída.

Del total computado como indisponibilidad para ese enlace en el período, **el 98,7% corresponde a este artefacto.**

**Mecanismo.** Con el servidor de monitoreo detenido no se realizan sondeos y ningún trigger puede dispararse: los problemas que habrían ocurrido durante la interrupción simplemente no existen. Pero un problema **ya abierto** no necesita datos para permanecer abierto; necesita datos para cerrarse. La condición de recuperación exige un valor nuevo que nunca llegó.

> **La interrupción del monitoreo sesga el indicador en una sola dirección.** Sobrecuenta la indisponibilidad de lo que ya estaba en falla al momento de la interrupción, y subcuenta a cero todo lo demás. No es ruido simétrico que se promedie: es sesgo sistemático.

Consecuencia: el indicador calculado sobre período calendario no es interpretable. El reporte de disponibilidad debe emitirse sobre período declarado.

#### 2. Ausencia de dato y supresión por dependencia son indistinguibles en consola

Un trigger dependiente que no aparece puede deberse a que la dependencia lo contuvo o a que nunca recibió el dato que lo dispara. Ambos casos producen la misma vista.

La opción de mostrar problemas suprimidos no resuelve la ambigüedad: en esta plataforma la supresión designa el efecto de una ventana de mantenimiento, no el de una dependencia. Una dependencia opera antes —el trigger no cambia de estado y no se crea evento— de modo que no hay nada que mostrar atenuado.

El criterio válido está en el historial del item, no en la lista de problemas.

#### 3. Fallo parcial: dispositivo vivo a gestión, inalcanzable al plano de datos

La condición construida para el experimento reproduce el patrón de fallo parcial: el dispositivo responde a consultas de gestión pero no a pruebas de alcance desde el servidor de monitoreo.

Un operador que interpreta la alerta de indisponibilidad como caída del dispositivo se equivoca. **Si hay telemetría fresca, el dispositivo está operativo y el fallo es de camino, filtro o política.** Esta verificación es la primera del runbook para ese trigger y ahorra al siguiente nivel la mitad del diagnóstico.

#### 4. La marca de última consulta no indica cuándo se consultó

Varios items estáticos figuraban sin actualizar durante más de una hora mientras los de intervalo corto reportaban al día. Una ejecución forzada se aceptó sin producir cambio.

Causa: un paso de preprocesamiento que descarta valores repetidos con intervalo de refresco de doce horas. El item se consulta según su intervalo y solo se almacena si el valor cambió.

**Interpretar un item "sin actualizar" como monitoreo roto es una causa frecuente de diagnóstico equivocado y de escalamiento innecesario.** El sistema opera según diseño.

Es la contraparte de un hallazgo previo de este proyecto —un host en estado saludable con telemetría nula—: allí la interfaz mostraba salud sin datos; aquí muestra silencio con recolección activa.

#### 5. La exclusión de período es un instrumento de reescritura del cumplimiento

Retira tiempo del cálculo completo, numerador y denominador. Su efecto **no es predecible sin conocer la distribución de las caídas**: puede mejorar el indicador, empeorarlo, o eliminarlo por completo.

En la prueba ejecutada el indicador **empeoró**, porque el tramo excluido era proporcionalmente más sano que el resto. Cuando la exclusión cubrió el período entero, el resultado fue "no aplicable".

> **Quien tenga permiso de edición sobre un acuerdo de nivel de servicio puede alterar el cumplimiento reportado sin modificar un solo dato de monitoreo, y el reporte no evidencia que se haya excluido algo.**

Tres consecuencias:

- **Gobernanza:** el permiso de edición de acuerdos de nivel de servicio es un control de integridad de reportes, no una función operativa. Debe separarse del rol que opera el centro de operaciones y ser auditado.
- **El resultado "no aplicable" es el comportamiento más honesto de la herramienta.** Excluir todo no produce cumplimiento total: produce ausencia de medición. El sistema se niega a emitir un número sin base.
- **Para el reporte:** el mecanismo es reversible y no destruye datos, pero su uso exige justificación documentada de cada exclusión. Declarar la contaminación en el texto sigue siendo la salida correcta, porque es auditable y la exclusión no lo es.

#### 6. Una interfaz permanentemente caída no genera alerta, y es correcto

Dos interfaces reportan estado caído de forma estable sin producir alarma. El trigger exige una transición de estado, no un estado.

Es el principio ya documentado en este proyecto —ausencia de alerta no equivale a fallo de detección— aplicado a la capa de red.

#### 7. La plataforma de monitoreo se alerta a sí misma

Durante el arranque simultáneo de varios sistemas, el servidor de monitoreo generó una alerta por saturación de sus propios procesos de sondeo.

**Una alerta del host de monitoreo se interpreta de forma distinta a una alerta de un host monitoreado:** indica presión sobre la recolección. Ignorarla implica que las lecturas posteriores dejan de ser confiables sin que nadie lo advierta.

#### 8. Editar un prototipo no actualiza los triggers ya generados

La descripción reescrita en el prototipo se propaga a los triggers descubiertos en el siguiente ciclo de descubrimiento automático, no de inmediato. Un operador que edita y prueba enseguida concluye que el cambio no se aplicó.

#### 9. Los secretos se filtran por las secciones narrativas de la documentación

En la documentación interna de una sesión previa, las credenciales estaban correctamente redactadas en todos los bloques de configuración y aparecían literales dentro de un ejemplo de comando erróneo incluido en la sección de aprendizajes.

> **La sanitización que revisa los bloques de configuración no es suficiente.** Los secretos se filtran por registros pegados, ejemplos de error y capturas de consola, donde nadie los busca porque no tienen forma de configuración. La revisión debe ejecutarse por búsqueda de patrón sobre el documento completo.

#### 10. El tiempo de detección es un parámetro de diseño

En la caída provocada, transcurrieron **2 minutos y 36 segundos** entre la falla real y la creación del evento. No es latencia accidental: es la consecuencia directa de exigir tres sondeos fallidos consecutivos sobre un item de un minuto, decisión explícita de intercambio entre falsos positivos y velocidad de detección.

> **Las métricas de tiempo de respuesta medidas desde la detección ocultan sistemáticamente ese tramo.** El servicio estuvo caído antes de que existiera la alerta. Un reporte que mide desde la detección presenta una realidad más benigna que la experimentada por el usuario.

El reporte de disponibilidad debe declarar ambos denominadores.

---

### Métricas del ciclo de incidente

| Hito | Δ desde falla real |
|---|---|
| Falla real | — |
| Detección | 2m 36s |
| Notificación entregada | 2m 39s |
| Asunción de responsabilidad con mensaje | 5m 13s |
| Resolución | 15m 36s |

Medidas desde la detección, las mismas magnitudes serían de 2m 37s y 13m. La diferencia entre ambas lecturas es el hallazgo 10.

---

### Limitaciones declaradas

- **El nodo emulado se reinicia espontáneamente**, sin causa registrada. Ocurrió tres veces durante esta sesión. Es un patrón conocido de virtualización anidada: riesgo de plataforma, no incidente de infraestructura, y fuente principal de contaminación de las métricas de disponibilidad.
- **El indicador de nivel de servicio calculado sobre período calendario no es interpretable**, por el sesgo asimétrico descrito en el hallazgo 1.
- **La dependencia de supresión se validó con una caída sintética**, construida mediante filtrado selectivo para aislar la variable. No es un escenario espontáneo.
- **Las métricas de tiempo de respuesta provienen de incidentes provocados y atendidos por el mismo operador.** No son métricas de operación: demuestran que el ciclo es medible desde el historial de la plataforma.
- **Los tiempos de escalamiento del runbook se definieron por criterio de impacto**, no derivados de un acuerdo de nivel de servicio contratado.
- **La caída de enlace se provoca por desactivación administrativa**, no por corte de medio físico. Limitación heredada de sesiones anteriores.
- **La migración de credenciales a macros de tipo secreto resuelve exposición en interfaz, interfaz de programación y exportación, no en almacenamiento.** El control apropiado para un entorno regulado sería un gestor externo de secretos.

---

### Entregables de la sesión

| Entregable | Estado |
|---|---|
| `runbooks/respuesta-triggers.md` | Completo — ocho triggers, matriz de escalamiento, anexo |
| Descripciones de trigger reescritas | Tres triggers de red, en origen |
| Credenciales en macros de tipo secreto | Migradas y validadas |
| Validación de dependencia de supresión | Ejecutada con diseño controlado |
| Experimento de reescritura de indicador | Concluido, cuatro intentos documentados |

El reporte de disponibilidad se traslada a la sesión siguiente. La regla fijada al abrir la sesión establecía que ese documento podía dividirse entre sesiones y el runbook no, porque un procedimiento de respuesta incompleto es peor que ninguno.

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*

---

## Sesión 4E — Corrección del modelo de servicio

**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Zabbix NOC · AustralPay Lab
Sesión: 4E

---

### Objetivo

Cerrar tres correcciones pendientes identificadas por el reporte de disponibilidad de la sesión anterior: la ausencia del dispositivo de red en el modelo de servicios, la denominación engañosa de un servicio de seguridad, y la verificación de la profundidad efectiva del escalamiento de notificaciones.

### Entorno

| Componente | Detalle |
|---|---|
| Plataforma de monitoreo | Zabbix 7.0.29 LTS sobre Docker Compose |
| Base de datos | PostgreSQL 16 |
| Dispositivo de red | Router Cisco IOSv monitoreado por SNMPv3, nivel `authPriv` |
| Hosts monitoreados | Dos servidores Linux, un endpoint Windows, un dispositivo de red |

### Estado inicial

Verificación previa a cualquier cambio. Dos problemas activos, ambos correspondientes a deuda técnica conocida y declarada; ninguno en la capa de red. El indicador de nivel de servicio del período cerrado publicado coincidió con cuatro decimales con el valor del reporte emitido, lo que habilita su uso como instrumento de control para el experimento de la sección 3.

**Reestimación declarada.** La primera corrección estaba estimada como un cambio de configuración menor. Consumió la sesión completa: exigió rediseño del criterio de selección de señal, edición de plantilla, un experimento de control sobre retroactividad y una validación con caída real. El empaquetado del repositorio se traslada a la sesión siguiente.

> La estimación falló por un supuesto, no por planificación. La tarea estaba descrita como "incorporar el dispositivo al árbol de servicios", formulación que omite la pregunta determinante: **por qué señal**. Un servicio no se enlaza a un host sino a etiquetas de eventos, y decidir cuáles es trabajo de modelado.

---

### Qué se hizo

#### 1. Selección de la señal para la capa de red

Se enumeraron los tipos de problema que el dispositivo genera, con su conjunto completo de etiquetas de fábrica, y se clasificaron según correspondan o no a indisponibilidad del servicio:

| Tipo de problema | Clasificación |
|---|---|
| Dispositivo inalcanzable | Indisponibilidad |
| Caída de enlace | Indisponibilidad |
| Pérdida elevada de paquetes | Degradación |
| Reinicio del dispositivo | Aviso |
| Interrupción de la recolección SNMP | Ceguera del monitoreo, no caída del servicio |

**Se descartó la vía de etiquetado a nivel de host.** El trigger de pérdida de paquetes lleva simultáneamente las etiquetas de alcance de disponibilidad y de rendimiento. Dado que las condiciones de un servicio se evalúan en conjunción y los operadores disponibles no incluyen negación, ningún filtro basado en esa taxonomía captura los dos triggers deseados excluyendo el tercero. Una degradación de un minuto habría computado como corte total del servicio de negocio.

**Vía adoptada: etiquetado selectivo en origen.** Se añadió una etiqueta propia al trigger de alcance del dispositivo y al prototipo de trigger de caída de enlace, usando en este último la macro de descubrimiento dentro del **valor** de la etiqueta.

El resultado es un conjunto de etiquetas con prefijo común, una por objeto descubierto, que un filtro por coincidencia parcial de prefijo captura con **una sola condición**.

**Costo declarado:** se edita la plantilla del fabricante, de modo que una actualización futura elimina las etiquetas. Deuda ya asumida en la sesión anterior por el mismo motivo.

**Ventaja de diseño:** el criterio de inclusión deja de depender de la taxonomía del fabricante y pasa a ser explícito. Un tipo de problema queda fuera porque no se etiquetó, no porque se haya inferido su clasificación.

Verificación: los cinco triggers presentaron su etiqueta con la macro resuelta, sin objetos sin cubrir.

#### 2. Creación del servicio de red

El servicio se ubicó **como hijo del servicio de negocio**, no de la raíz del árbol. El dispositivo transporta el tráfico entre la aplicación y la base de datos; su caída interrumpe el procesamiento, y esa dependencia debe estar en el modelo. Ubicarlo en la raíz lo habría hecho visible sin propagar al servicio de negocio: presencia decorativa y persistencia del defecto original.

**Costo aceptado:** el indicador del servicio de negocio empeorará hacia adelante, al incorporar un componente de laboratorio con reinicios espontáneos. Un indicador que mejora al agregar un componente frágil estaría mal calculado.

Se detectó en la verificación —no en el diseño— que el servicio requería además la etiqueta de selección del objeto de acuerdo de nivel de servicio, distinta de las etiquetas de problema. Sin ella el servicio propaga estado correctamente pero no obtiene línea propia en el reporte de cumplimiento.

#### 3. Control de retroactividad del indicador

**Pregunta:** ¿modificar la estructura del árbol reescribe el indicador de períodos ya cerrados y publicados?

**Diseño:** lectura de línea base inmediatamente previa al cambio, cambio, relectura sobre el mismo período cerrado.

**Resultado:** los tres indicadores publicados permanecieron idénticos a cuatro decimales. El reporte emitido conserva su validez. El cálculo se apoya en los eventos de estado ya registrados de cada servicio, que no se regeneran al modificar la estructura.

**Resultado adicional no buscado:** el servicio creado figura como **no aplicable** en los períodos anteriores a su existencia, no como cumplimiento total. La plataforma se niega a emitir cumplimiento sobre un período que no midió.

Se reconfirmó además el artefacto de período abierto documentado en la sesión anterior: el indicador de la semana en curso subió sin tratamiento alguno, únicamente por crecimiento del denominador con el reloj.

#### 4. Validación con caída real

Se desactivó administrativamente el enlace de datos, manteniendo alcanzable la interfaz de gestión. El trigger de alcance del dispositivo no dispara en esa condición, de modo que la prueba aísla específicamente la cobertura a nivel de interfaz, que es lo único que la vía adoptada agrega sobre la descartada.

**Primer intento:** el servicio permaneció en estado correcto durante más de cinco minutos con el problema activo. Se verificaron operador del filtro, etiqueta en el trigger y etiqueta en el evento —los tres correctos— antes de formular causa.

**Segundo intento**, con el servicio recreado y el problema ya activo: transición inmediata y propagación completa.

Configuración idéntica, resultados distintos. Se declara como deuda técnica con hipótesis de latencia de sincronización del gestor de servicios y su prueba escrita. **No se afirma causa raíz.**

**Propagación validada de extremo a extremo:** los tres niveles del árbol —hoja, rama y raíz— transicionaron al estado del problema, con la causa raíz propagada íntegra hasta el nivel superior. Restitución verificada en los tres niveles.

#### 5. Corrección de padre duplicado

Durante la validación se detectó que el servicio nuevo colgaba simultáneamente de la raíz y del servicio de negocio.

**Causa:** la plataforma precarga el campo de servicios padre con el servicio desde el que se navega al pulsar la creación. El campo llega con contenido y se lee como vacío si no se inspecciona.

Con la raíz como padre directo, una caída de red la alcanza por un camino que **omite** el servicio de negocio: el impacto queda descrito dos veces por rutas distintas y se pierde la lectura que justifica un árbol de servicios. Corregido a padre único.

#### 6. Denominación del servicio de seguridad

Servicio renombrado conforme a la recomendación del reporte de disponibilidad. El identificador interno no cambia, de modo que historial e indicador se conservan.

**Residual declarado:** en el árbol el nombre se interpreta correctamente porque el padre aporta contexto, pero el reporte de cumplimiento lista los servicios en plano y ahí subsiste la ambigüedad. Un prefijo explícito lo resolvería en ambas vistas; no se aplicó.

#### 7. Verificación del escalamiento

Configuración leída sin modificar: tres operaciones, cada una en un paso único, con inicio inmediato, a los cinco minutos y a los diez minutos respectivamente.

**La escalada termina en el minuto diez.** Un problema sin reconocer más allá de ese punto no genera notificación adicional porque no hay operación configurada que la genere. La lectura de "tres pasos" de una sesión previa era correcta en número e incompleta en implicancia.

---

### Hallazgos

#### 1. Una etiqueta de fábrica puede pertenecer a dos categorías simultáneamente

El trigger de pérdida de paquetes lleva a la vez las etiquetas de alcance de disponibilidad y de rendimiento. Combinado con la ausencia de operador de negación en las condiciones de un servicio, eso **inhabilita esa taxonomía como criterio de selección** para alimentar un indicador de disponibilidad.

> **Las taxonomías de etiquetas provistas por el fabricante están diseñadas para filtrar vistas, no para definir denominadores contractuales.** Un indicador construido sobre ellas mide lo que el fabricante decidió agrupar, no lo que la organización acordó medir.

Es la contraparte del hallazgo de un bloque previo, donde se rechazó reutilizar la misma etiqueta para los servicios de aplicación y base de datos por estar contaminada con los triggers de salud del agente de monitoreo. El mismo defecto en dos capas distintas.

#### 2. La macro de descubrimiento en el valor de la etiqueta produce un filtro estable

Etiquetar un prototipo con un valor que contiene la macro de descubrimiento genera etiquetas distintas por objeto, que un filtro por prefijo captura con una condición única.

El patrón sobrevive los ciclos de descubrimiento: los objetos nuevos nacen etiquetados sin intervención sobre el servicio. Es el mismo principio que resolvió la supresión por dependencia en una sesión anterior —actuar sobre el prototipo y no sobre el objeto generado—, aplicado ahora a la selección de señal.

#### 3. El campo de servicios padre se precarga con el contexto de navegación

Al crear un servicio desde dentro de otro, el campo llega con ese servicio ya cargado y sin aviso.

> Pertenece a la misma familia que otros hallazgos de este proyecto: **el sistema realiza una acción por el operador y no la declara.** La contramedida es constante en todos los casos: verificar el estado del objeto después de la acción en lugar de confiar en la acción.

#### 4. Modificar la estructura del árbol no reescribe el cumplimiento histórico

Junto con el resultado del experimento de la sesión anterior, queda cerrado el mapa de qué puede alterar un número ya publicado:

| Acción | ¿Reescribe el pasado? |
|---|---|
| Ventana de mantenimiento retroactiva | No |
| Modificación de la estructura del árbol de servicios | No |
| Exclusión de período sobre el objeto de acuerdo | **Sí** |

**Una sola palanca reescribe el cumplimiento, y es la que no deja rastro en el reporte.** Eso concentra el problema de gobernanza en un permiso único y lo vuelve auditable: el permiso de edición sobre objetos de acuerdo de nivel de servicio es un control de integridad de reportes, no una función operativa.

#### 5. El identificador del canal de notificación se propaga como etiqueta automática

Cada evento que originó notificación incorpora una etiqueta generada por el sistema que contiene el identificador del canal de destino. No la escribe ningún operador: la añade el mecanismo de notificación para correlacionar el mensaje enviado.

> **Amplía el hallazgo de sanitización de la sesión anterior.** Allí los secretos se filtraban por secciones narrativas de la documentación. Aquí se filtran por **metadatos generados por el propio sistema**, visibles en cualquier captura de consola que expanda etiquetas. La revisión por búsqueda de patrón sobre texto no es suficiente: debe cubrir también el material gráfico, y exige conocer de antemano qué patrón buscar.

#### 6. El escalamiento repite antes de escalar, y termina

Las dos primeras operaciones notifican al mismo destinatario: es repetición, no escalamiento. El único cambio de nivel real es el tercero.

No existe operación recurrente. Un paso definido con rango abierto se repetiría hasta el reconocimiento; sin él, el sistema asume que si nadie respondió en diez minutos, la notificación cumplió su función.

> El incidente sin reconocer durante 74 minutos documentado en el reporte de disponibilidad demuestra que ese supuesto es falso. **Es la corrección de fondo que ese reporte solicitaba sin poder nombrarla**, y queda como recomendación explícita.

#### 7. Discrepancia entre pasos configurados y notificaciones registradas

El reporte de disponibilidad registra dos notificaciones para el incidente espontáneo; hay tres operaciones configuradas. Falta la dirigida al grupo de segundo nivel.

Existe precedente en un bloque anterior: un permiso denegatorio interrumpiendo la escalada sin dejar traza en el registro. Se declara como deuda con esa hipótesis y su prueba, no como conclusión.

---

### Limitaciones declaradas

- **La caída de validación se provoca por desactivación administrativa** de la interfaz, no por corte de medio físico. Limitación heredada de sesiones previas.
- **La deuda de latencia de sincronización queda sin causa raíz.** Se declara con hipótesis y prueba escrita; la evidencia del episodio original se perdió al resolver el problema antes de documentarlo.
- **El etiquetado del prototipo alcanza a todas las interfaces descubiertas.** La caída de cualquiera computa contra el indicador del servicio de negocio. Decisión de alcance deliberada: toda caída de enlace es indisponibilidad dura, criterio con el que se excluyó la degradación por pérdida de paquetes.
- **Las etiquetas aplicadas residen en la plantilla del fabricante** y una actualización de esa plantilla las elimina.
- **El indicador del período en curso queda contaminado** por la caída de validación de esta sesión. No afecta al reporte publicado, que cubre un período cerrado anterior.
- **El servicio de red mide alcance del dispositivo y estado de interfaces.** No mide degradación, reinicios ni interrupción de la recolección, por decisión de diseño documentada.

---

### Entregables de la sesión

| Entregable | Estado |
|---|---|
| Capa de red incorporada al árbol de servicios | Completo — validada de extremo a extremo con caída real |
| Etiquetado selectivo en trigger y prototipo | Aplicado; cinco objetos cubiertos |
| Control de retroactividad del indicador | Ejecutado; reporte publicado confirmado válido |
| Denominación del servicio de seguridad | Corregida; residual declarado |
| Verificación de profundidad del escalamiento | Completa; genera recomendación y una deuda nueva |

El empaquetado del repositorio se traslada a la sesión siguiente. El criterio de corte fue el habitual del proyecto: se prefiere una corrección validada de extremo a extremo sobre dos tareas iniciadas y ninguna cerrada.

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.*
