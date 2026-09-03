# Estado del proyecto y deuda técnica

> AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.

**Estado:** cerrado. Seis sesiones entre el 5 y el 17 de agosto de 2026.

Este es el documento donde está lo que no funcionó, lo que quedó a medias y lo que se decidió no hacer. Se publica completo por criterio: un repositorio de laboratorio sin deuda declarada no es un repositorio sin deuda, es uno que no la registró.

---

## 1. Registro de deuda

| # | Ítem | Estado |
|---|---|---|
| 1 | Operaciones de actualización no generan envío | **Cerrado como declarado** — intermitente, 7 hipótesis refutadas con evidencia directa, alcance acotado |
| 5 | Desfase de zona horaria entre frontend y registro del servidor | Documentado — recomendación emitida en el reporte |
| 6 | Descripciones de trigger en idioma distinto | **Cerrado** para los triggers de red |
| 7 | Disco de la estación sobre 90% | Aceptado |
| 8 | Host por defecto del servidor se auto-reactiva | Mitigado en el muro con grupos explícitos |
| 11 | Base de datos medida a nivel TCP | Declarado |
| 12 | Puerto de base de datos abierto a toda la red local | **Pendiente** |
| 13 | Endpoint de salud creado pero no consultado | Cosmético |
| 15 | Auto-registro sin clave precompartida | Deshabilitado tras la prueba |
| 16 | Problemas reconocidos escalan indefinidamente | Mitigado |
| 17 | Severidad uniforme en ambos enlaces | Identificado, no implementado |
| 18 | Descripciones de trigger explicaban la expresión lógica | **Cerrado** — reescritas en origen |
| 19 | Credenciales SNMPv3 en texto literal | **Cerrado** — macros de tipo secreto, validadas contra el export |
| 20 | Interfaces ausentes del inventario | **Cerrado** — enumeración directa |
| 23 | Dependencia de supresión sin validar | **Cerrado** — validada con telemetría viva |
| 24 | Convención de nombres inconsistente en grupos de host | Identificado |
| 25 | Ruta al runbook duplicada en la notificación | Identificado |
| 26 | Editar la plantilla del fabricante se pierde ante una actualización | Declarado |
| 27 | Borde de red fuera del árbol de servicios | **Cerrada** — corregida y validada con caída real de extremo a extremo |
| 28 | Servicio de endpoint denominado como si midiera disponibilidad | **Cerrada** — renombrado; residual de lectura en vista plana declarado |
| 29 | Profundidad efectiva del escalamiento sin verificar | **Cerrada** — configuración leída; genera recomendación y deuda #31 |
| 30 | Servicio recién creado no evalúa un problema ya activo | **Abierta** — hipótesis de latencia de sincronización. Prueba: crear servicio, esperar 10 min sin tocar nada, provocar caída |
| 31 | Tercer paso del escalamiento sin notificación registrada en el incidente del 17-08 | **Abierta** — hipótesis: permiso del grupo de segundo nivel sobre el grupo de hosts. Prueba: `Users → User groups → Host permissions` |
| **32** | Passphrases SNMPv3 expuestas en capturas de trabajo durante la ejecución | **Abierta, prioridad alta** — las capturas publicadas están tachadas; corresponde rotar las credenciales en el equipo y en las macros |

**Balance:** ocho cerradas, una cerrada como declarada, tres abiertas, el resto identificadas o aceptadas con criterio explícito.

---

## 2. Recomendaciones emitidas y no implementadas

**Paso recurrente en el escalamiento.** Agregar un paso con rango abierto para que un problema no reconocido siga generando notificación indefinidamente. El incidente espontáneo del 17 de agosto estuvo 1h 14m sin acuse de recibo y las notificaciones se agotaron antes que el incidente.

**Severidad diferenciada por enlace** (deuda #17). Ambos enlaces del equipo de borde alertan con la misma severidad pese a tener criticidad distinta.

**Deduplicar la ruta al runbook** en la plantilla de notificación (deuda #25).

---

## 3. Lo que no está probado

Se lista aparte de la deuda porque no es trabajo pendiente sino **límite de lo afirmado**.

- La deuda #30 y la #31 están planteadas como **hipótesis**, no como causas. Ninguna se verificó.
- El experimento de retroactividad de la ventana de mantenimiento sobre el indicador se ejecutó una vez, sin repetición.
- El escalamiento se validó en las dos direcciones, pero el tercer paso solo se observó registrado en pruebas provocadas, no en el incidente espontáneo.
- **La marca de tiempo de una captura registra cuándo se tomó, no cuándo el estado mostrado era verdadero.** Ninguna captura de `evidence/` se presenta como prueba de secuencia temporal; la cronología de los hechos está reconstruida en los informes desde el histórico de ítems y el registro del servidor, que sí tienen hora fiable.

---

## 4. Evidencia no preservada

De las 56 capturas comprometidas en los informes, 19 no sobrevivieron al archivo de trabajo. Están listadas una por una, con lo que mostraba cada una, en [`../evidence/README.md`](../evidence/README.md).

Se declaran en vez de omitirse. El hallazgo que cada una respaldaba está documentado en el informe de su sesión; lo que falta es el respaldo gráfico, no el hallazgo.

---

## 5. Cierre

El proyecto se cierra con el modelo de servicio corregido y validado, el runbook operativo, el reporte de disponibilidad emitido sobre período declarado, y tres deudas abiertas con su procedimiento de verificación escrito.

No se cierra porque no quede nada por hacer. Se cierra porque lo que queda está identificado, acotado y escrito.
