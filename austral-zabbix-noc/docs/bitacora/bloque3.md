# Bloque 3 — Servicios de negocio, SLA, mantenimiento, escalamiento y auto-registro (Zabbix NOC)

**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Zabbix NOC · AustralPay Lab
Sesiones: 3A, 3B, 3C

---

## Objetivo

Elevar el monitoreo desde el plano de la máquina al plano del servicio de negocio: definir servicios con SLA medible, controlar el ruido temporal mediante ventanas de mantenimiento, garantizar que ninguna alerta quede sin dueño mediante escalamiento, y automatizar el onboarding de hosts.

## Entorno

| Componente | Detalle |
|---|---|
| Stack | Zabbix 7.0.29 LTS sobre Docker Compose |
| Servidor | `austral-noc` |
| Hosts monitoreados | `austral-app`, `austral-db` (Linux), `austral-ws01` (Windows) |
| Servicios de aplicación | nginx (capa de aplicación), PostgreSQL 16 (capa de datos) |
| Canal de notificación | Webhook nativo de mensajería instantánea |

## Estado inicial

Al cierre del Bloque 2 el monitoreo cubría disponibilidad de host y alertado con notificación entregada, pero sin un modelo de servicio de negocio. Tres carencias definieron el trabajo:

- La disponibilidad se medía por ICMP, que responde desde la pila de red del kernel: un servidor puede responder ping mientras la aplicación está muerta o el puerto cerrado.
- Los hosts de aplicación y base de datos no ejecutaban ningún daemon real. El escenario que justifica un NOC —un proceso que muere sin que la máquina caiga— no podía producirse.
- Las notificaciones se enviaban una sola vez. Un incidente nocturno sin lector permanecía sin dueño hasta la mañana siguiente.

---

## Qué se hizo

### Capa de servicio: de la máquina al negocio

**1. Instalación de servicios reales.** nginx en el host de aplicación (con página de presentación y endpoint de salud) y PostgreSQL en el host de datos. El motor de base de datos escuchaba por defecto solo en loopback, condición que habría producido un falso positivo permanente indistinguible de una caída real; corregido mediante `ALTER SYSTEM`, que escribe en un archivo de sobrescritura en vez de modificar la configuración base, y restringido a la dirección de LAN en lugar de todas las interfaces.

**2. Verificación de alcance desde el contenedor del servidor**, previa a cualquier configuración. La verificación desde el host anfitrión no es válida: el contenedor tiene su propia pila de red. La imagen oficial no incluye utilidades de red por diseño de superficie mínima, por lo que se empleó el builtin de shell correspondiente.

**3. Items de medición mediante Simple check**, ejecutados desde el servidor y no desde el agente:

| Host | Chequeo | Capa |
|---|---|---|
| Aplicación | HTTP sobre puerto 80 | L7 |
| Base de datos | TCP sobre puerto 5432 | L4 |

La elección del tipo de item es determinante para el cálculo de SLA: un chequeo ejecutado por el agente deja de reportar cuando el host cae (ausencia de dato), mientras que un sondeo externo devuelve un cero explícito. El SLI requiere el segundo, porque mide lo que mide el cliente.

**4. Triggers de medición dedicados**, en severidad `Warning` de forma deliberada —por debajo del umbral de la acción de notificación— con tags exclusivos por servicio.

La severidad expresa aquí **función y no criticidad**: el trigger alimenta el indicador de nivel de servicio sin generar notificación, separando el canal de medición del canal de alertado. La decisión fue necesaria porque el tag de disponibilidad de las plantillas oficiales está compartido por tres triggers de naturaleza distinta, y la caída del agente de monitoreo no constituye indisponibilidad del servicio de negocio sino pérdida de la capacidad de observarlo. Los triggers heredados de plantilla no admiten re-etiquetado a nivel de host.

**5. Tag a nivel de host** en el endpoint Windows, inyectado automáticamente por Zabbix en todos los problemas que ese host genere, presentes y futuros, sin modificar plantillas.

**6. Árbol de servicios de tres niveles:**

```
AustralPay                                   [Most critical of child services]
├── Procesamiento de Pagos
│   ├── Servidor de aplicación
│   └── Base de datos transaccional
└── Cobertura de controles de seguridad
    └── Endpoint austral-ws01
```

La regla de cálculo de estado es una declaración de arquitectura, no una preferencia de configuración: `Most critical of child services` describe dependencias en serie sin redundancia, que es la topología real del laboratorio. La alternativa (`Most critical if all children have problems`) describiría un conjunto con balanceo.

La segunda rama mide **cobertura de controles de seguridad** — el porcentaje del tiempo en que la telemetría de seguridad del endpoint estuvo efectivamente operando. Es una métrica de auditoría real (*agent health* / cobertura de EDR): un control desplegado en el 100% de los endpoints pero operando el 91% del tiempo deja una ventana ciega del 9%. Su inclusión hace que el NOC mida al SOC, materializando en un número reportable la premisa de infraestructura compartida entre ambos proyectos.

Se reservó una rama para el dispositivo de red, pendiente de reincorporación, de modo que entre sin rediseñar el árbol.

**7. Dos objetos SLA simétricos:**

| | Procesamiento de Pagos | Cobertura de Controles de Seguridad |
|---|---|---|
| SLO | 99.9% | 99% |
| Período | Semanal | Semanal |
| Zona horaria | Explícita | Explícita |
| Ventana de servicio | 7 días, 09:00–23:00 | 7 días, 09:00–23:00 |

Objetivos distintos por criterio explícito: un reinicio autorizado del endpoint detiene sus servicios de seguridad de forma legítima, por lo que el objetivo persigue patrones de ausencia y no cada arranque. Aplicar la misma vara a ambos servicios sería un error de criterio.

Ambos SLA se configuraron con ventana y zona horaria idénticas. Un desalineamiento inicial —un hueco horario de una hora en uno de ellos— habría producido una ventana de medición ciega no declarada y, además, denominadores distintos que invalidan la comparación entre ambos indicadores.

**8. Validación del matching** mediante detención controlada del servidor web. Se comprobaron cuatro cosas distintas: la captura del problema por coincidencia de tags (visible en la columna de causa raíz), la propagación jerárquica de hoja a raíz, el aislamiento entre ramas, y la ausencia total de notificación durante la degradación.

### Capa de ruido temporal: ventanas de mantenimiento

**9. Ventana con recolección activa** sobre el endpoint Windows, con filtrado por tags.

La modalidad con recolección se prefiere en operación: la modalidad sin recolección deja un hueco de datos indistinguible de un fallo de monitoreo en una investigación posterior, y constituye una ventana ciega de telemetría. Desde la perspectiva de seguridad, un período de mantención sin recolección es un período en el que la actividad no genera evidencia.

**El filtro por tags es la decisión de diseño del componente.** Una ventana que suprime todo suprime también las alertas de seguridad, y el mantenimiento programado es precisamente el momento en que la detención de servicios resulta esperada y no se cuestiona. El filtro se configuró de modo que los problemas de disponibilidad y rendimiento se supriman y los de seguridad no.

El campo de tags opera como lista blanca de supresión y no admite operador de negación, por lo que la exclusión se expresa enumerando lo que sí debe suprimirse. La limitación es conservadora y se declara: un tag nuevo no queda suprimido por defecto, es decir, el sesgo del sistema es alertar de más y nunca silenciar de más.

**10. Tres escenarios validados:**

| Escenario | Resultado |
|---|---|
| Caída de disponibilidad durante la ventana | Problema suprimido, visible solo con el filtro correspondiente. Sin notificación |
| Detención de un control de seguridad durante la ventana | Problema visible sin filtro. Notificación de severidad alta con procedimiento de respuesta y mapeo MITRE ATT&CK entregada |
| Efecto sobre el indicador de nivel de servicio | Ver hallazgos |

### Capa de responsabilidad: escalamiento

**11. Grupo de usuarios de segundo nivel** con permiso de lectura sobre los hosts —mínimo necesario para ver un problema y acusar recibo, sin capacidad de reconfiguración— y usuario asociado con su propio canal.

**12. Acción convertida en escalada de tres pasos**, con intervalo de cinco minutos:

| Paso | Destino | Condición |
|---|---|---|
| 1 | Operador de primer nivel | — |
| 2 | Operador de primer nivel | Evento sin acuse de recibo |
| 3 | **Grupo** de segundo nivel | Evento sin acuse de recibo |

El tercer paso se dirige a un grupo y no a un usuario: un turno de segundo nivel es un rol con varias personas, y la incorporación de un integrante nuevo no debe requerir modificar la acción.

**El intervalo se derivó del presupuesto de error del objetivo de nivel de servicio, no de un valor arbitrario.** Con un SLO de 99.9% semanal sobre la ventana declarada, el margen es de aproximadamente nueve minutos por semana; un escalamiento de treinta minutos sería decorativo, porque cuando el segundo nivel se enterara el objetivo ya estaría incumplido varias veces.

El escalamiento introduce además una consecuencia operativa sobre el acuse de recibo: deja de ser una marca de lectura y pasa a significar asunción de responsabilidad, porque detiene la cadena.

**13. Validación en dos direcciones.** Primero la escalada completa —tres notificaciones en secuencia, la tercera con formato diferenciado y macro de tiempo transcurrido sin atención— y después su interrupción por acuse de recibo, confirmando que el tercer paso no se ejecuta.

La segunda prueba es la más relevante: un escalamiento que no se detiene cuando el incidente ya tiene dueño es peor que no tenerlo.

### Capa de inventario: auto-registro

**14. Acción de auto-registro** con condición sobre metadatos del agente y tres operaciones: alta del host, asignación a grupo y enlace de plantilla.

El auto-registro se eligió frente al descubrimiento activo de red por dos razones: no genera tráfico de sondeo —indistinguible de un reconocimiento hostil y motivo de incidente en una red vigilada— y el agente se identifica mediante metadatos, de modo que la plantilla correcta se asigna sin necesidad de inferirla. Su limitación es simétrica: solo detecta lo que tiene agente instalado, por lo que no sustituye al descubrimiento de red como control de inventario frente a activos no autorizados. En producción ambos mecanismos coexisten con propósitos distintos.

Se prefirieron metadatos explícitos sobre metadatos derivados de un item del sistema: más verbosos, pero no dependen de un formato de salida que puede cambiar entre versiones del sistema operativo y romper el matching sin señal visible.

**15. Prueba sin pérdida de historial.** En lugar de eliminar un host productivo, se modificó temporalmente el identificador del agente de uno de los servidores Linux. El host apareció en el inventario de forma automática con su grupo y plantilla correspondientes, generando 57 items y 25 triggers sin intervención manual. Revertido y eliminado tras la verificación.

La acción se dejó **deshabilitada** al terminar: sin clave precompartida, el auto-registro acepta a cualquier agente que conozca la dirección del servidor, lo que permitiría contaminar el inventario.

### Diagnóstico de detección de agente

**16.** Se investigó por qué la detención prolongada del agente del endpoint no había generado alerta. La hipótesis inicial —una dependencia circular en la que el agente sería responsable de reportar su propia ausencia— resultó incorrecta: el item que sustenta ese trigger es interno y lo calcula el servidor.

La causa real es que la condición nunca llegó a cumplirse: la macro de tiempo de espera es de cinco minutos y la expresión evalúa un mínimo sobre esa ventana; en cada ocasión el período se interrumpió antes de completarse. El informe de triggers más frecuentes confirmó cero eventos, descartando que se tratara de un problema disparado y suprimido.

Una prueba controlada de seis minutos con el host operativo confirmó el funcionamiento correcto en las tres capas: item en estado no disponible, problema generado en severidad alta, notificación entregada y recuperación registrada.

---

## Hallazgos

### El mantenimiento excluye del cálculo de disponibilidad de forma automática y sin registro

Un control de seguridad detenido durante una ventana de mantenimiento activa generó problema y notificación —el filtro por tags funcionó según lo diseñado— y sin embargo el indicador de nivel de servicio del endpoint permaneció en 100%, con tiempo de indisponibilidad en cero y la columna de exclusiones declaradas vacía. Un servicio comparable sin ventana activa registró la caída equivalente y bajó de su objetivo.

| | Filtro de tags de la ventana | Cálculo del indicador |
|---|---|---|
| Problemas de disponibilidad | Suprime notificación | Excluye |
| Problemas de seguridad | **No** suprime — la notificación se entregó | **Excluye igual** |

Son dos mecanismos independientes: el filtro por tags gobierna la notificación, mientras que la exclusión del cálculo depende únicamente de que el host se encuentre bajo mantenimiento. Adicionalmente, el tiempo disponible medido aumentó tras la ventana, lo que indica que el período no se retiró del denominador sino que se contabilizó como tiempo de servicio correcto.

**Consecuencia operativa:** quien administra las ventanas de mantenimiento gobierna la cifra reportada, sin modificar el objeto SLA y sin dejar rastro en el reporte entregable. Es la razón por la que en operaciones formales las ventanas requieren aprobación y preaviso: no son una comodidad del equipo de operaciones sino un instrumento con consecuencia contractual.

**Recomendación derivada:** el reporte de disponibilidad no se entrega de forma aislada. Debe acompañarse del listado de ventanas de mantenimiento del período; sin él la cifra es cierta pero incompleta.

### El denominador domina el indicador

Mediciones sucesivas sobre los mismos eventos arrojaron cifras distintas sin que cambiara nada salvo el tiempo transcurrido. La misma interrupción de dos minutos representa una degradación severa sobre un período de treinta minutos y un incumplimiento marginal sobre una semana completa.

El presupuesto de error mostró el mismo comportamiento, creciendo a medida que avanzaba el período: no es una cuota fija sino una fracción del tiempo medido.

**Un indicador de disponibilidad sin su denominador declarado no es interpretable.**

### Un permiso denegado interrumpe el escalamiento sin dejar rastro

El tercer paso de la escalada no se ejecutó en dos incidentes distintos, sin que apareciera ninguna operación fallida en el registro de acciones: no hubo error, ni reintento, ni estado de fallo. La causa era un permiso denegado del grupo de segundo nivel sobre el grupo de hosts.

Zabbix resuelve el conjunto de destinatarios filtrando por permisos **antes** de generar el envío. Un destinatario sin acceso al host no constituye un envío fallido sino un destinatario que no aplica, y por ello no genera registro. Los dos primeros pasos funcionaban porque estaban dirigidos a una cuenta de superadministración, que accede a todos los hosts sin evaluar permisos.

**Un escalamiento puede permanecer roto de forma indefinida sin que nadie lo advierta, porque la única señal es la ausencia de un mensaje que nadie está esperando.** La verificación no puede ser visual: exige ejecutar la escalada completa y confirmar recepción en cada nivel.

Es la cuarta manifestación del mismo filtro implícito de permisos documentado en el Bloque 2, allí como riesgo teórico.

### Ausencia de alerta no equivale a fallo de detección

Un componente aparentemente no detectado durante días resultó ser un escenario que nunca llegó a producirse: la ventana de evaluación se interrumpía en cada ocasión antes de completarse.

**Antes de diagnosticar el detector es necesario verificar que la condición llegó a cumplirse.** El informe de triggers más frecuentes permite distinguir "disparó y fue suprimido" de "nunca disparó", que exigen diagnósticos opuestos.

### Un filtro de supresión invertido es indistinguible de uno correcto

El campo de tags de las ventanas de mantenimiento no admite operador de negación. Expresar la intención con el operador disponible más cercano produce el comportamiento exactamente contrario —suprimir seguridad y dejar sonando disponibilidad— sin generar error, advertencia ni registro.

**Cada capa de filtrado se valida disparando un evento de cada categoría, no revisando la pantalla de configuración.**

### Un problema aceptado como limitación escala de forma indefinida

Una alerta correspondiente a una limitación declarada del entorno, que por decisión no se remedia, completó la escalada hasta el segundo nivel por sí sola. Confirma el funcionamiento del mecanismo sobre un problema no provocado, pero constituye un antipatrón: escalar de forma recurrente algo que nadie va a resolver entrena al segundo nivel a ignorar los escalamientos.

Mitigado mediante acuse de recibo con justificación explícita registrada en el propio incidente, lo que además deja el criterio auditable para quien revise el histórico. Un acuse sin mensaje detiene la escalada igual pero no explica nada, y un incidente sin explicación es indistinguible de uno abandonado.

### Item huérfano de descubrimiento alertando sobre un componente inexistente

Un servicio descubierto automáticamente en su momento dejó de existir tras una actualización del sistema, y el item asociado continuó generando alertas indicando literalmente que el servicio no existe. Un item huérfano de descubrimiento de bajo nivel alerta de forma indefinida sobre algo ausente.

Mitigado deshabilitando el trigger; la vía de fondo es la política de recursos perdidos en la regla de descubrimiento o la exclusión por macro.

### Exposición de metadatos del transporte en las plantillas de mensaje

La macro que vuelca todos los tags de un evento incorporaba identificadores internos generados por el propio canal de notificación, sin significado para el operador. Sustituida por macros selectivas de los tres tags con valor de triaje. Ante un tag ausente, la macro selectiva devuelve un valor explícito de desconocido, comportamiento que se acepta como preferible a omitir el campo: distingue "sin mapeo" de "no evaluado".

---

## Limitaciones declaradas

- **Los indicadores obtenidos en este bloque no constituyen métricas de disponibilidad representativas.** Corresponden a períodos de medición cortos con interrupciones inyectadas de forma deliberada para validar el modelo, y a un entorno de laboratorio que no opera de forma continua. Se documentan como validación funcional del módulo, no como resultado operativo.
- **Ventana de operación declarada.** El laboratorio no opera 24x7. En un entorno productivo de procesamiento de pagos la ventana sería continua y el objetivo de nivel de servicio, superior.
- **Escalamiento con destino físico compartido.** La topología de objetos —grupos, usuarios y canales independientes— es real; el destino final de notificación es común por limitación del entorno.
- **Medición de la capa de datos a nivel de transporte.** El sondeo valida aceptación de conexión, no ejecución de consultas. La ruta de mejora existe mediante el plugin nativo correspondiente, a costa de un usuario de monitoreo dedicado y modificación de la configuración de autenticación.
- **Auto-registro sin clave precompartida.** Deshabilitado tras la validación. En producción requeriría cifrado con clave precompartida.

## Estado del checklist

- [x] Servicios de aplicación y datos reales instalados y verificados desde el contenedor del servidor
- [x] Items de sondeo externo en capas L7 y L4
- [x] Triggers de medición con severidad funcional y tags exclusivos
- [x] Árbol de servicios de tres niveles con regla de cálculo justificada
- [x] Matching de problemas a servicios validado con interrupción controlada
- [x] Dos objetos SLA simétricos con ventana, fecha efectiva y zona horaria explícitas
- [x] Reporte de disponibilidad generado e interpretado
- [x] Ventana de mantenimiento con filtrado por tags
- [x] Supresión de disponibilidad validada
- [x] No supresión de seguridad validada
- [x] Efecto del mantenimiento sobre el cálculo de disponibilidad verificado
- [x] Escalada de tres niveles configurada y validada
- [x] Interrupción de la escalada por acuse de recibo validada
- [x] Auto-registro con metadatos validado de extremo a extremo
- [x] Detección de indisponibilidad de agente diagnosticada y probada
- [x] Entorno devuelto a estado limpio

## Próximo paso

Bloque 4 — dashboard de operaciones, registro documentado del ciclo completo de una caída simulada, runbook de respuesta por trigger y reporte de disponibilidad mensual. Repetición de la captura del reporte de SLA sobre un período extendido, para contrastar el efecto del denominador contra la medición inicial.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
