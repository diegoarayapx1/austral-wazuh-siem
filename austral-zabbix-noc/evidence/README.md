# Índice de evidencia — Proyecto 2 · AustralPay NOC (Zabbix)

> **AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.** Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.

Este índice describe cada captura del proyecto: qué muestra y por qué se conservó. Las capturas están organizadas por sesión, en subcarpetas `bloqueN/`.

---

## Convenciones

**Direccionamiento.** El laboratorio opera sobre direccionamiento privado RFC 1918. Las capturas lo muestran tal como fue capturado, sin edición. La documentación en `docs/` normaliza el direccionamiento al rango de documentación RFC 5737 (`192.0.2.0/24`) por legibilidad. La diferencia entre ambos es deliberada y está declarada aquí: un rango privado de laboratorio no constituye información sensible, y editar los píxeles de una captura para ocultarlo destruiría el contexto que la hace verificable.

**Credenciales.** Ninguna captura publicada expone credenciales. Dos capturas mostraban las passphrases SNMPv3 en texto plano y se publican con esos campos tachados; el tachado está indicado en la tabla correspondiente. Las credenciales del laboratorio fueron rotadas con posterioridad.

**Numeración.** La numeración es la declarada en el informe técnico de cada sesión, de modo que una captura citada en un informe se encuentra por su nombre. La separación en subcarpetas por sesión resuelve las colisiones de numeración entre sesiones sin alterar los nombres.

**Capturas no preservadas.** Algunas capturas comprometidas en los informes no sobrevivieron al archivo de trabajo. Están listadas con su descripción original y marcadas como no preservadas. Se declaran en vez de omitirse: un índice que oculta sus huecos no es verificable.

**Lo que la hora del archivo no dice.** La marca de tiempo del archivo registra el momento de la captura, no el momento en que el estado mostrado era verdadero. Una pestaña del navegador cargada antes conserva el estado de su carga. Por eso ninguna captura de este índice se rotula por posición en una secuencia; se rotula por lo que muestra. La cronología de los hechos está en los informes, reconstruida desde el historial de ítems y el registro del servidor, que sí son fuentes con hora fiable.

---

## bloque1 — Instalación y alta de hosts

| Archivo | Qué muestra |
|---|---|
| `01-hosts-activos.png` | Los tres hosts con agente en estado disponible, con conteo de ítems recolectando en cada uno |

---

## bloque2 — Triggers, dependencias y notificaciones

| Archivo | Qué muestra |
|---|---|
| `01-icmp-firewall-antes-despues.png` | Sondeo ICMP a la estación Windows: 100% de pérdida antes de la regla de firewall, 0% después. Causa y resolución en una sola captura |
| `02-regla-firewall-icmp.png` | Creación de la regla con alcance mínimo: solo eco ICMP entrante, y solo desde el servidor de monitoreo |
| `05-supresion-dependencia-icmp.png` | Lista de problemas con problemas suprimidos visibles: una sola alerta activa pese a que el host está caído por completo. La dependencia funciona |
| `07-media-severidades.png` | Media del usuario con filtro de severidad: las tres bajas en gris, las tres altas activas. Destino de notificación tachado |
| `09-ciclo-problema-recuperacion.png` | Notificación de problema y de recuperación encadenadas por identificador de problema original, con duración calculada |
| `10-action-log.png` | Registro de acciones de un ciclo completo: problema, envío, resolución, envío de recuperación, todos en estado enviado |

**No preservadas:** `03-triggers-seguridad-high.png` (los cuatro triggers del stack de seguridad en severidad alta con mapeo MITRE) · `04-items-seguridad-latest-data.png` (los cuatro ítems recolectando) · `06-dependencia-configurada.png` (la dependencia declarada en el hijo) · `08-alerta-telegram-problema.png` (notificación con plantilla personalizada y bloque de acción esperada).

---

## bloque3 — Servicios, SLA, mantenimiento y escalamiento

| Archivo | Qué muestra |
|---|---|
| `04-sla-reporte.png` | Reporte de cumplimiento del servicio de pagos: rama y dos hojas, indicador por debajo del objetivo |
| `05-sla-cobertura-seguridad.png` | Reporte de cumplimiento del servicio de seguridad, con su objetivo propio |
| `06-arbol-servicios.png` | Raíz del árbol de servicios con sus dos ramas y la causa raíz propagada |
| `06b-arbol-rama-pagos.png` | Segundo nivel del árbol. Va con la anterior: el árbol tiene tres niveles y una sola pantalla no los muestra |
| `08-seguridad-atraviesa-mantenimiento.png` | **Central.** Notificación de manipulación de controles de seguridad recibida con una ventana de mantenimiento activa sobre el host |
| `08b-seguridad-atraviesa-mantenimiento-consola.png` | La contraparte en consola: ícono de mantenimiento en el host y el problema de seguridad igualmente visible. El filtro por tag hace lo que se diseñó |
| `09-sla-exclusion-mantenimiento.png` | Reporte de cumplimiento con indicador en 100, indisponibilidad en cero y el campo de exclusiones **vacío**. Vale por lo que no muestra: la ventana no alimentó ese campo |
| `10-escalamiento-telegram.png` | Los tres pasos del escalamiento en secuencia, con la marca de nivel 2 y la duración sin atención |
| `11-escalamiento-detenido-por-ack.png` | Registro de acciones: pasos uno y dos enviados, reconocimiento del operador, paso tres ausente. Contraste con la anterior |

**No preservadas:** `07-mantenimiento-suprime-disponibilidad.png` (problema de disponibilidad suprimido durante la ventana) · `12-auto-registro.png` (host incorporado automáticamente con plantilla enlazada) · `13-falso-positivo-mitre.png` (notificación con clasificación contradictoria — el falso positivo documentado como deuda).

---

## bloque4a — Capa de red y SNMPv3

| Archivo | Qué muestra |
|---|---|
| `17-snmpv3-zabbix-aes128.png` | Ventana de prueba devolviendo la identificación del sistema con cifrado AES128 desde el servidor de monitoreo. **Passphrases tachadas** |
| `18-linkdown-problem.png` | Caída de enlace en estado de problema en consola, con hora de inicio |
| `19-linkdown-telegram.png` | Notificación de la caída recibida en el canal |
| `20-linkdown-operstatus.png` | Estado operativo de la interfaz en caído, con dato fresco |
| `21-lld-filtro-override.png` | Macro de filtro de descubrimiento sobrescrita, con el valor heredado visible al lado |
| `22b-engineid-fallo-autenticacion.png` | Interfaz SNMP no disponible con error de autenticación, en un host cuyas credenciales no cambiaron. Es el incidente de regeneración del identificador de motor visto desde la consola |
| `23-lld-trigger-borrado-programado.png` | Trigger generado por descubrimiento marcado para borrado tras dejar de ser descubierto |
| `24-topologia-eve-ng.png` | Topología emulada: el segmento de prueba desconectado del equipo de borde. Es el enlace sobre el que se provocan las caídas |
| `25-items-deshabilitados-vios.png` | Sensores de temperatura, fuentes y ventiladores deshabilitados: hardware que la plantilla del fabricante espera y el equipo emulado no tiene |
| `26-items-snmp-sin-datos.png` | Los ítems de recolección SNMP fallando en bloque, sin último valor |

**No preservadas:** `14-snmpv3-walk.png` (recorrido de interfaces por SNMPv3 con credenciales tachadas) · `15-snmpv3-acl-denegada.png` (timeout desde un origen no autorizado, con el contador de denegaciones y el registro del equipo) · `16-snmp-verde-sin-datos.png` (**la más significativa del bloque**: indicador de disponibilidad SNMP en verde con el 100% de los ítems fallando) · `22-engineid-no-snmp-collection.png` (el trigger de ausencia de recolección disparado y resuelto).

> `26-items-snmp-sin-datos.png` cubre la mitad del argumento de `16`: los ítems fallando. La otra mitad —el indicador de disponibilidad en verde al mismo tiempo— no está en ninguna captura conservada. El hallazgo está documentado en el informe del bloque; su prueba gráfica, no.

---

## bloque4b — Operación NOC y ciclo de incidente

| Archivo | Qué muestra |
|---|---|
| `05-muro-noc.png` | Muro de operación completo: problemas activos, conteo por severidad, disponibilidad de hosts y cumplimiento por servicio en una sola pantalla |
| `06-trigger-problem.png` | Problema activo sin reconocer, en el instante de la detección |
| `07-incidente-reconocido.png` | El mismo incidente reconocido a los 27 segundos, con el mensaje del operador en el registro de acciones |
| `08-ciclo-resuelto.png` | Incidente resuelto con hora de recuperación y duración calculada |
| `09-ciclo-action-log.png` | Registro completo del ciclo: notificación, reconocimiento, recuperación, con horas |
| `10-dependencia-prototipo.png` | Los cuatro prototipos de trigger de interfaz con su dependencia declarada. Declarada en el prototipo, sobrevive a los ciclos de descubrimiento |
| `11-telegram-ciclo.png` | Problema y recuperación en el canal, con la plantilla corregida |
| `13-pruebas-instrumentadas-deuda1.png` | Registro de acciones con las pruebas numeradas por el propio operador durante el diagnóstico del fallo de notificación |

> Las tres primeras del ciclo sostienen las métricas del informe: detección, reconocimiento a los 27 segundos, resolución al minuto. Las cifras se pueden verificar contra la consola en vez de creerse.

**No preservada:** `12-contraste-escalamiento.png` (contraste de operaciones: escalada completa sin reconocimiento frente a escalada detenida por reconocimiento).

---

## bloque4c — Validaciones, secretos y runbook

| Archivo | Qué muestra |
|---|---|
| `09-dependencia-supresion-problems.png` | Una sola alerta en la lista de problemas con el trigger maestro activo. La supresión validada con telemetría viva |
| `10-dependencia-supresion-latest-data.png` | Estado operativo de la interfaz en caído con dato de menos de un minuto. Va con la anterior: prueba que hay dato fresco y aun así no hay segunda alerta |
| `11b-macros-referenciadas-en-host.png` | Los tres campos de credenciales SNMPv3 del host resueltos por referencia a macro |
| `11c-credenciales-antes-migracion.png` | La misma pantalla antes de la migración, con los valores literales. **Passphrases tachadas** |
| `14-event-details-descripcion-runbook.png` | Detalle de evento con la descripción reescrita como procedimiento y **la ruta al runbook dentro del propio trigger**. El procedimiento llega al operador en la alerta, no en el repositorio |

> `11c` y `11b` son la misma pantalla antes y después de la migración a macros de tipo secreto. Con `11c` tachada, el par muestra el problema y su corrección sin exponer nada.

**No preservadas:** `11-macros-secret-text.png` (las tres macros con valor oculto en la pestaña de macros) · `12-experimento-sli-retroactivo.png` (indicador en sus tres estados: limpio, excluido y no medido) · `13-telegram-accion-esperada.png` (notificación con procedimiento accionable).

---

## bloque4d — Incidente espontáneo y reporte de cumplimiento

| Archivo | Qué muestra |
|---|---|
| `14-caida-espontanea-sin-operador.png` | Caída del borde de red de 1h 14m, **sin reconocimiento en ningún momento**. El único incidente del proyecto que no fue provocado |
| `16-sli-calendario-vs-declarado.png` | Reporte de cumplimiento con tres semanas consecutivas y sus denominadores distintos. Muestra por qué el indicador en período calendario no es interpretable en este laboratorio |
| `17-arbol-servicios-sin-red.png` | **El hallazgo principal del proyecto.** El servicio de negocio con sus dos hijos: no incluye el borde de red. El indicador reportaba cumplimiento normal con el borde caído porque el borde no pertenecía a ningún servicio |
| `17b-raiz-sin-capa-red.png` | La raíz del árbol en el mismo estado. Va con la anterior: una muestra que falta el servicio, la otra que falta la rama entera |
| `18-icmp-caida-espontanea.png` | Gráfico de sondeo del período, con el promedio de la ventana visible en la leyenda |

**No preservada:** `15-escalamiento-sin-ack.png` (registro de acciones del incidente: notificaciones enviadas, ningún reconocimiento).

---

## bloque4e — Corrección del modelo de servicio

| Archivo | Qué muestra |
|---|---|
| `15-tags-trigger-prototipo.png` | Etiqueta de servicio en el prototipo de trigger de enlace, con la macro de nombre de interfaz sin resolver |
| `15b-tag-network-core.png` | Etiqueta de servicio en el trigger de alcance del equipo. Es la otra mitad del etiquetado selectivo en origen |
| `18-arbol-caida-propagada-raiz.png` | Raíz del árbol degradada |
| `18b-arbol-caida-propagada-rama.png` | Servicio de negocio degradado con la causa raíz propagada desde la red |
| `18c-arbol-caida-propagada-hoja.png` | El servicio de red degradado, con la causa raíz identificada por interfaz |
| `20-arbol-restituido.png` | El árbol con la capa de red incorporada y todos los servicios en estado correcto |
| `21-renombramiento-antes.png` | El servicio de seguridad con su denominación anterior, que sugería medir disponibilidad |
| `21b-renombramiento-despues.png` | El mismo servicio renombrado, conservando su identificador interno y por lo tanto su historial |

> Las tres capturas de propagación son el mismo fenómeno en los tres niveles del árbol. Por separado ninguna prueba propagación: prueban que tres servicios están degradados. Juntas muestran la causa raíz recorriendo el árbol de la hoja a la raíz.
>
> `17-arbol-servicios-sin-red.png` (bloque4d) y `18b` son el par que sostiene el hallazgo y su corrección. Se leen en ese orden.

**No preservadas:** `16-tags-triggers-descubiertos.png` (los triggers generados con su etiqueta resuelta por interfaz) · `17-sli-linea-base-post-cambio.png` (indicadores idénticos tras el cambio estructural, con el servicio nuevo sin medición previa) · `19-evento-tag-sla.png` (evento con su etiqueta de servicio resuelta).

---

## Resumen

| Concepto | Cantidad |
|---|---|
| Capturas declaradas en los informes | 56 |
| Declaradas y conservadas | 36 |
| Declaradas y cubiertas por otra captura del índice | 1 |
| Declaradas y no preservadas | 19 |
| Capturas adicionales incorporadas al índice | 16 |
| **Archivos publicados** | **52** |

Las diecinueve no preservadas están descritas en su sección correspondiente. El hallazgo que cada una respaldaba está documentado en el informe técnico de su bloque; lo que falta es su respaldo gráfico, no el hallazgo.

---

*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.*
