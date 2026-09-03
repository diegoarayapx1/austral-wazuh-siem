# austral-zabbix-noc

> **AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.** Ninguna infraestructura, dato, incidente o métrica de este repositorio corresponde a una empresa real. Este trabajo no fue realizado para ningún cliente.

Monitoreo NOC sobre Zabbix 7.0 LTS: despliegue, modelo de servicio con objetivos de nivel de servicio, monitoreo SNMPv3 de un equipo de red emulado, escalamiento de incidentes con notificación, runbook de respuesta y reporte de disponibilidad.

Laboratorio personal, ejecutado en seis sesiones entre el 5 y el 17 de agosto de 2026.

---

## Qué hay acá

| Ruta | Contenido |
|---|---|
| **[`docs/informe-final-proyecto2.md`](docs/informe-final-proyecto2.md)** | **Informe final del proyecto. Empezar acá.** |
| [`docs/arquitectura.md`](docs/arquitectura.md) | Inventario, topología y modelo de monitoreo |
| [`docs/despliegue.md`](docs/despliegue.md) | Cómo está desplegado el stack y con qué decisiones |
| [`docs/snmp-config.md`](docs/snmp-config.md) | SNMPv3 authPriv contra el equipo de borde, y el control de acceso que lo protege |
| [`docs/sla-definicion.md`](docs/sla-definicion.md) | Árbol de servicios, objetivos, ventana de medición y qué mide cada indicador |
| [`docs/estado-y-deuda-tecnica.md`](docs/estado-y-deuda-tecnica.md) | Estado al cierre y registro completo de deuda técnica |
| [`docs/bitacora/`](docs/bitacora/) | Informe técnico por bloque |
| [`runbooks/respuesta-triggers.md`](runbooks/respuesta-triggers.md) | Procedimiento de respuesta por trigger |
| [`reports/`](reports/) | Reporte de disponibilidad de período declarado |
| [`templates/`](templates/) | Export del host de red |
| [`evidence/`](evidence/) | Capturas, indexadas y descritas |

---

## Los tres hallazgos que sostienen el proyecto

**El árbol de servicios reportaba cumplimiento normal con el borde de red caído.** El indicador no mentía: medía correctamente un modelo incompleto. El equipo de borde no pertenecía a ningún servicio, así que su caída no tenía por dónde propagarse. Corregido, validado con una caída real de extremo a extremo, y documentado con el antes y el después.
→ [`bloque4.md`](docs/bitacora/bloque4.md) · `evidence/bloque4d/17-arbol-servicios-sin-red.png`

**Ausencia de alerta no es lo mismo que detección funcionando.** El indicador de disponibilidad SNMP se mantuvo en verde mientras el total de los ítems de recolección fallaba. El equipo respondía al sondeo de alcance y no a las consultas reales.
→ [`bloque4.md`](docs/bitacora/bloque4.md)

**Los fallos silenciosos son los caros.** Un permiso mal puesto rompió un paso del escalamiento sin dejar rastro en ningún registro. Un identificador que el equipo regenera al arrancar invalidó credenciales que nadie tocó. Ninguno de los dos produjo error.
→ [`bloque4.md`](docs/bitacora/bloque4.md)

---

## Sobre el direccionamiento

El laboratorio opera sobre direccionamiento privado RFC 1918. **Las capturas de `evidence/` lo muestran tal como fue capturado**; la documentación de `docs/` lo normaliza al rango de documentación RFC 5737 (`192.0.2.0/24`) por legibilidad.

La diferencia es deliberada y se declara acá para que no se lea como descuido. Editar los píxeles de una captura para ocultar un rango privado de laboratorio destruiría el contexto que la hace verificable, sin proteger nada: `192.168.1.0/24` no es información sensible. Lo que sí lo era —credenciales SNMPv3 visibles en dos capturas— está tachado, y su rotación está declarada como deuda abierta en [`estado-y-deuda-tecnica.md`](docs/estado-y-deuda-tecnica.md).

---

## Cómo leerlo

Si tenés diez minutos: [`docs/informe-final-proyecto2.md`](docs/informe-final-proyecto2.md). Está todo ahí, sin referencias a evidencia.

Si te interesa el criterio más que el resultado: [`docs/estado-y-deuda-tecnica.md`](docs/estado-y-deuda-tecnica.md). Es el documento donde está lo que no funcionó, lo que quedó a medias y lo que se decidió no hacer.

---

## Stack

Zabbix 7.0.29 LTS (Server, Frontend nginx) · PostgreSQL 16 · Docker Compose · Ubuntu Server 24.04 LTS · EVE-NG con Cisco vIOS · VMware Workstation
