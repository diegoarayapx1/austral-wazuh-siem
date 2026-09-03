# Informe final — Proyecto 2: NOC sobre Zabbix

> **AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.** Ninguna infraestructura, dato, incidente o métrica de este informe corresponde a una empresa real. Este trabajo no fue realizado para ningún cliente.

**Período:** 5 al 17 de agosto de 2026 · cuatro bloques, nueve sesiones de trabajo
**Stack:** Zabbix 7.0.29 LTS · PostgreSQL 16 · Docker Compose · Ubuntu Server 24.04 LTS · Cisco vIOS 15.9(3) sobre EVE-NG
**Estado:** cerrado, con tres deudas abiertas y su procedimiento de verificación escrito

---

## 1. Qué se construyó

Un centro de operaciones de red funcional sobre una infraestructura simulada de cinco nodos: tres servidores con agente, una estación de trabajo y un equipo de borde monitoreado por SNMPv3.

El sistema hace cuatro cosas distintas, y la distinción importa porque cada una falla de forma diferente:

**Detecta.** Triggers sobre métricas de sistema, servicio y red, con dependencias que suprimen las alertas derivadas de una causa común.

**Notifica.** Escalamiento de tres pasos con plantilla propia por tipo de evento, incluyendo el procedimiento de respuesta dentro de la propia notificación.

**Mide.** Árbol de servicios de tres niveles con dos objetivos de nivel de servicio y ventana de medición declarada, alimentado por triggers de medición separados del canal de alertado.

**Documenta.** Runbook de respuesta por trigger, reporte de disponibilidad sobre período declarado, y bitácora técnica de cada sesión.

---

## 2. Recorrido por bloques

### Bloque 1 — Despliegue

Servidor de monitoreo sobre VM dedicada, stack Zabbix en Docker Compose con versión fijada, y tres hosts reportando en modo activo.

Las decisiones que sobrevivieron todo el proyecto se tomaron acá: versión fija en lugar de `latest`, credenciales en archivo de entorno separado del compose, agentes con versión anclada al servidor, y modo activo de forma consistente.

Primer fallo silencioso del proyecto: la instalación desatendida del agente en Windows termina sin instalar nada y sin devolver error cuando falta elevación.

### Bloque 2 — Triggers, dependencias y notificación

Alta del equipo de borde, triggers por severidad, dependencias entre triggers y canal de notificación con plantilla propia.

Tres cosas de fondo:

**El sondeo ICMP requirió una regla de firewall explícita** en el host Windows, con alcance mínimo —solo eco entrante, solo desde el servidor de monitoreo— en lugar de "permitir ICMP". Un host que bloquea el eco es indistinguible de un host caído.

**Las dependencias entre triggers se declaran en el prototipo, no en el trigger generado.** Los triggers creados por descubrimiento se recrean en cada ciclo; una dependencia declarada sobre uno de ellos se pierde en la siguiente ejecución sin dejar rastro.

**El canal de notificación es un patrón documentado de comando y control.** Se usa en el laboratorio por costo cero, y se declara que en un entorno con requisitos de cumplimiento no sería una elección aceptable. La decisión se documenta con su límite, no se presenta como recomendación.

### Bloque 3 — Servicios, objetivos, mantenimiento y escalamiento

El bloque donde el monitoreo pasa de reportar máquinas a reportar servicios.

**Árbol de servicios de tres niveles** con regla de cálculo `Most critical of child services`, elegida porque describe dependencias en serie sin redundancia —la topología real— y no por preferencia de configuración.

**Dos objetivos con criterio distinto**, 99.9% para el servicio de pagos y 99% para la cobertura de controles de seguridad, con ventana y zona horaria idénticas. Un desalineamiento de una hora entre ambos habría producido denominadores distintos que invalidan cualquier comparación.

**Triggers de medición en severidad por debajo del umbral de notificación.** La severidad expresa función y no criticidad: el trigger alimenta el indicador sin alertar. Fue necesario porque la etiqueta de disponibilidad de las plantillas oficiales está compartida por tres triggers de naturaleza distinta, y la caída del agente de monitoreo no es indisponibilidad del servicio: es pérdida de la capacidad de observarlo.

**Ventana de mantenimiento con recolección activa y filtrado por etiquetas.** Una ventana que suprime todo suprime también las alertas de seguridad — y el mantenimiento programado es precisamente el momento en que la detención de un servicio de seguridad resulta esperada y no se cuestiona. Se configuró para que disponibilidad y rendimiento se supriman y seguridad no.

**Escalamiento de tres pasos con intervalo derivado del presupuesto de error.** Con 99.9% semanal sobre la ventana declarada el margen es de unos nueve minutos; un escalamiento de treinta sería decorativo. El tercer paso apunta a un grupo y no a una persona.

La validación que importa no fue que el escalamiento escalara, sino que **se detuviera al recibir acuse**. Un escalamiento que sigue corriendo cuando el incidente ya tiene dueño es peor que no tenerlo.

### Bloque 4 — Capa de red, operación y corrección del modelo

El bloque más largo y el que produjo los hallazgos.

**SNMPv3 en nivel authPriv** contra el equipo de borde, con lista de control de acceso debajo. Dos capas con funciones distintas: la ACL previene, el cifrado protege lo que pasó la ACL, y el registro de denegaciones convierte un intento fallido en evidencia.

**Credenciales migradas a macros de tipo secreto**, verificado contra el export: las macros salen declaradas sin valor. Es una propiedad comprobable contra un artefacto, no una afirmación.

**Muro de operación** con seis widgets, y ciclo completo de incidente instrumentado: detección, reconocimiento a los 27 segundos, resolución al minuto.

**Descripciones de trigger reescritas como procedimiento**, con la ruta al runbook dentro del propio trigger. El procedimiento llega al operador en la alerta, no en el repositorio.

**Corrección del árbol de servicios**, que es el hallazgo principal y tiene sección propia más abajo.

---

## 3. Los cuatro hallazgos

### 3.1 El indicador reportaba cumplimiento normal con el borde de red caído

Durante un incidente no provocado, el equipo de borde estuvo inalcanzable 1h 14m y el indicador de nivel de servicio no se movió.

**El indicador no mentía.** Medía correctamente un modelo incompleto: el equipo de borde no pertenecía a ningún servicio del árbol, así que su caída no tenía por dónde propagarse. La métrica era exacta y la conclusión que sugería era falsa.

Es el hallazgo más transferible del proyecto porque el error no está en la herramienta ni en la configuración, sino en la correspondencia entre el modelo y la infraestructura. Un objetivo de nivel de servicio mide lo que se le declaró que mida. Si lo declarado es incompleto, el número es correcto e inútil al mismo tiempo.

Corregido incorporando la conectividad de red como servicio hijo, con etiquetado selectivo en origen, y validado provocando una caída real y observando la propagación de hoja a raíz.

### 3.2 Ausencia de alerta no es lo mismo que detección funcionando

El indicador de disponibilidad SNMP del equipo se mantuvo en verde mientras el total de sus ítems de recolección fallaba.

La causa: el indicador de disponibilidad responde al sondeo de alcance, y el equipo respondía. Las consultas reales fallaban por otro motivo. Dos señales distintas que la interfaz presenta juntas.

La consecuencia operativa es la que vale: **una consola sin alertas rojas no prueba que el monitoreo esté funcionando.** Prueba que no hay alertas. Verificar recolección efectiva —última fecha de recolección, último valor— es una tarea distinta de mirar el semáforo.

### 3.3 Los fallos silenciosos son los caros

Cuatro casos independientes del mismo patrón a lo largo del proyecto:

| Fallo | Cómo se manifiesta |
|---|---|
| Instalación desatendida sin elevación en Windows | Termina, no instala, no devuelve error |
| Permiso de grupo mal configurado | Rompe un paso del escalamiento sin línea en ningún registro |
| Identificador de motor SNMP regenerado al arrancar | Invalida credenciales que nadie tocó; el síntoma acusa contraseña incorrecta |
| Filtro de descubrimiento por defecto | Desactiva el trigger de una interfaz al apagarla, tratándola como recurso perdido |

Ninguno produce un error interpretable. Los cuatro se encontraron buscando la ausencia de algo que debía estar, no leyendo un mensaje de error. Es la diferencia entre monitorear y verificar.

### 3.4 El indicador en período calendario no es interpretable en un laboratorio intermitente

El entorno no corre de forma continua: se enciende para trabajar. El denominador de cada semana depende de cuántas horas estuvo encendido, así que dos semanas consecutivas no son comparables y ninguna representa disponibilidad real.

La salida fue emitir el reporte sobre **período declarado** en lugar de mensual, con una sección explícita de contaminación de la medición. Un número mensual habría sido más presentable y falso.

---

## 4. Método

Cuatro prácticas que se sostuvieron a lo largo del proyecto y que explican los hallazgos mejor que cualquier decisión técnica puntual.

**Verificación de línea base antes de cada cambio.** Cada sesión abrió con un inventario del estado: qué problemas están activos, cuántos ítems tiene cada host, qué contenedores corren. Sin eso, cualquier problema que aparezca durante la sesión es ambiguo entre preexistente y causado.

**Refutación con evidencia directa en lugar de conjetura.** El diagnóstico del fallo de notificación intermitente refutó siete hipótesis contra la base de datos antes de reclasificarlo. Se cerró como intermitente con alcance acotado, no como resuelto.

**Cierre acotado con declaración honesta por encima de resolución forzada.** Tres deudas quedan abiertas con su hipótesis y su procedimiento de prueba escritos. Ninguna se presenta como causa: están declaradas como hipótesis porque no se verificaron.

**Los errores de método se documentan como lección, no se corrigen en silencio.** Ejemplo del cierre: al ordenar las capturas del árbol de servicios apareció una secuencia imposible —degradado, correcto, degradado, en cincuenta y dos segundos—. La conclusión fue que **la marca de tiempo de una captura registra cuándo se tomó, no cuándo el estado mostrado era verdadero**, y que la cronología debe reconstruirse desde el histórico de ítems y el registro del servidor. Quedó escrito en el índice de evidencia.

---

## 5. Métricas

| Métrica | Valor | Origen |
|---|---|---|
| Tiempo hasta reconocimiento, ciclo instrumentado | 27 s | Registro de acciones |
| Tiempo hasta resolución, ciclo instrumentado | 60 s | Registro de acciones |
| Duración del incidente no provocado | 1h 14m | Consola, confirmado contra histórico |
| Reconocimientos durante ese incidente | 0 | Registro de acciones |
| Hosts monitoreados | 5 | — |
| Deudas cerradas | 8 + 1 acotada | Registro de deuda |
| Deudas abiertas al cierre | 3 | Registro de deuda |

Las dos primeras corresponden a un incidente provocado y observado por un operador que sabía que iba a ocurrir. **No son representativas de operación real** y se declaran como tales: miden que el instrumento registra correctamente el ciclo, no la capacidad de respuesta de un turno.

---

## 6. Lo que este laboratorio no demuestra

- **Escala.** Cinco hosts y un equipo de red. El descubrimiento automático se justifica con decenas de interfaces; con dos, ítems explícitos serían igualmente claros. Se usó por realismo operativo, no por necesidad.
- **Alta disponibilidad.** Una sola instancia, sin respaldo automatizado. Un fallo del servidor es pérdida total del histórico.
- **Cifrado de transporte.** Ni entre agente y servidor, ni en el frontend. En producción sería requisito.
- **Operación continua.** El entorno se enciende para trabajar, lo que contamina toda medición de período calendario.
- **Turno real.** Un solo operador, que además construyó el sistema y sabía dónde iba a fallar.

---

## 7. Estado al cierre

El proyecto cierra con el modelo de servicio corregido y validado contra una caída real, el runbook operativo enganchado a los triggers, el reporte de disponibilidad emitido sobre período declarado, y el registro de deuda completo y público.

Tres deudas quedan abiertas: la latencia de sincronización de un servicio recién creado, la anomalía del tercer paso del escalamiento durante el incidente no provocado, y la rotación de las credenciales SNMPv3 expuestas en capturas de trabajo. Las tres con hipótesis y procedimiento de verificación escritos.

No cierra porque no quede nada por hacer. Cierra porque lo que queda está identificado, acotado y escrito.

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.*
