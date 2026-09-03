# Arquitectura

> AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.
>
> El direccionamiento de este documento usa el rango de documentación RFC 5737 (`192.0.2.0/24`). Las capturas de `evidence/` muestran el direccionamiento privado real del laboratorio. Ver la nota del `README.md` de la raíz.

---

## 1. Inventario

| Host | Rol | Dirección | SO | Modo |
|---|---|---|---|---|
| `austral-noc` | Servidor de monitoreo | `192.0.2.70` | Ubuntu Server 24.04 LTS | — |
| `austral-app` | Servidor de aplicación (nginx) | `192.0.2.55` | Ubuntu | Agente activo |
| `austral-db` | Base de datos transaccional (PostgreSQL 16) | `192.0.2.60` | Ubuntu | Agente activo |
| `austral-ws01` | Estación de operaciones | `192.0.2.65` | Windows | Agente activo |
| `austral-net` | Equipo de borde (Cisco vIOS) | `192.0.2.75` | IOS 15.9(3) | SNMPv3 |

Los cuatro primeros corren sobre VMware Workstation en modo puente. El equipo de red está emulado en EVE-NG, que a su vez corre como máquina virtual sobre el mismo hipervisor.

**Monitoreo en modo activo** en los tres hosts con agente. Es el modo preferido a escala: reduce carga en el servidor y tolera hosts detrás de NAT o firewall. Se aplicó de forma consistente desde el inicio, no como corrección posterior.

---

## 2. Topología del equipo de borde

| Interfaz | Índice | Propósito | Estado |
|---|---|---|---|
| `Gi0/0` | 1 | LAN de gestión | Alcanzable desde el servidor de monitoreo |
| `Gi0/1` | 2 | Segmento interno aislado | Enlace de prueba para simulación de caída |
| `Gi0/2`, `Gi0/3` | 3, 4 | — | Administrativamente caídas |

**Por qué dos interfaces y no una.** Con una sola, la única forma de simular una caída sería apagar el equipo — y en ese escenario el monitoreo pierde también el SNMP: se observaría "host inalcanzable", no "interfaz caída". Con una segunda interfaz hacia un segmento aislado, la caída del enlace ocurre **mientras el equipo sigue respondiendo y contando lo que pasó**. Son dos eventos distintos y el monitoreo debe distinguirlos.

**Persistencia de identificadores.** Se fijaron dos parámetros que el equipo regenera al arrancar y que el monitoreo asume estables: el índice de interfaces y el identificador de motor SNMP. Sin persistencia del índice, una interfaz puede cambiar de posición tras un reinicio y todos los ítems quedan apuntando a la interfaz equivocada **sin producir ningún error**. Ver [`snmp-config.md`](snmp-config.md).

---

## 3. Grupos de host

| Grupo | Miembros |
|---|---|
| `AustralPay Servers` | `austral-app`, `austral-db`, `austral-ws01` |
| `AustralPay/Red` | `austral-net` |

La convención es inconsistente entre ambos —uno con espacio, otro con barra— y está declarada como deuda #24. Se documenta en vez de corregirse en silencio: en un entorno con reglas de permisos por grupo, renombrar un grupo tiene efectos que exceden lo cosmético.

---

## 4. Modelo de monitoreo

**Plantillas oficiales del fabricante**, sin modificar. Editar una plantilla del fabricante hace que la próxima actualización sobrescriba el cambio o genere conflicto; está declarado como deuda #26. Cuando hizo falta desviarse del comportamiento por defecto se usaron macros a nivel de host, que sí sobreviven a la actualización.

**Patrón de ítem maestro con ítems dependientes** en el equipo de red: una única consulta recorre la tabla de interfaces completa y una regla de descubrimiento parsea ese bloque para crear los ítems individuales, que extraen su valor sin generar tráfico adicional. El motivo es de escala —un equipo de 48 puertos con ocho métricas por puerto serían cientos de consultas por ciclo frente a una— y tiene una consecuencia práctica: hasta que el ítem maestro recolecta y el descubrimiento se ejecuta, **no existe ningún ítem de interfaz**.

**Dependencias entre triggers** para suprimir alertas derivadas. Cuando un host cae por completo, la alerta que importa es la de alcance; las de sus servicios son ruido. La dependencia se declara **en el prototipo de trigger, no en el trigger generado**: los triggers generados por descubrimiento se recrean en cada ciclo y una dependencia declarada sobre uno de ellos se pierde.

---

## 5. Muro de operación

Dashboard global con seis widgets: problemas activos, conteo por severidad, hosts con problemas, disponibilidad de hosts y dos reportes de cumplimiento —uno por objeto SLA.

**Zabbix 7.0 no exporta dashboards globales desde el frontend** —solo los de plantilla— así que no hay artefacto importable de este componente. Su registro es `evidence/bloque4b/05-muro-noc.png` y esta descripción. Reproducirlo requiere recrear los widgets a mano o consultarlo por API.

Los widgets se configuraron con grupos de host explícitos y no con "todos": el host por defecto del servidor de monitoreo se auto-reactiva tras algunas operaciones y contaminaba los conteos (deuda #8).

---

## 6. Notificación

Canal de mensajería con plantilla propia por tipo de evento. Cada notificación incluye host, severidad, hora de inicio, clasificación, componente, técnica MITRE cuando aplica, y un bloque de **acción esperada** con el procedimiento y la ruta al runbook.

La decisión de fondo: una alerta que solo dice qué se rompió obliga al operador a buscar qué hacer. La alerta lleva el procedimiento encima.

> **Nota sobre el canal.** El servicio de mensajería usado en el laboratorio es un patrón documentado de comando y control y exfiltración. En un entorno con requisitos de cumplimiento —PCI-DSS, por ejemplo— no sería una elección aceptable para alertado operativo. Se usa acá por costo cero y porque el objetivo del componente es demostrar el diseño de la plantilla y del escalamiento, no el transporte.
