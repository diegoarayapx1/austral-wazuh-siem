# Modelo de servicio y objetivos de nivel de servicio

> AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.

---

## 1. Árbol de servicios

```
AustralPay                                   [Most critical of child services]
├── Procesamiento de Pagos
│   ├── Servidor de aplicación
│   ├── Base de datos transaccional
│   └── Conectividad de red (austral-net)     ← incorporado en la sesión 4E
└── Cobertura de controles de seguridad
    └── Estación de operaciones
```

**La regla de cálculo es una declaración de arquitectura, no una preferencia.** `Most critical of child services` describe dependencias en serie sin redundancia, que es la topología real del laboratorio. La alternativa —`Most critical if all children have problems`— describiría un conjunto con balanceo, y afirmarla sobre esta topología sería falso.

La captura de problemas por servicio funciona por **coincidencia de etiquetas**, no por host: el servicio declara qué etiqueta le corresponde y absorbe cualquier problema que la traiga. Esto permite que un mismo host alimente servicios distintos según qué se rompa en él.

---

## 2. Objetivos

| | Procesamiento de Pagos | Cobertura de Controles de Seguridad |
|---|---|---|
| Objetivo | 99.9% | 99% |
| Período | Semanal | Semanal |
| Zona horaria | Explícita | Explícita |
| Ventana de servicio | 7 días, 09:00–23:00 | 7 días, 09:00–23:00 |

**Objetivos distintos por criterio explícito.** Un reinicio autorizado de la estación detiene sus servicios de seguridad de forma legítima; el objetivo del segundo servicio persigue patrones de ausencia, no cada arranque. Aplicar la misma vara a ambos sería un error de criterio, no una simplificación.

**Ventana y zona horaria idénticas en ambos.** Un desalineamiento inicial —un hueco de una hora en uno de ellos— habría producido una ventana de medición ciega no declarada y, peor, denominadores distintos que invalidan cualquier comparación entre los dos indicadores.

---

## 3. Qué mide la segunda rama

La rama de seguridad mide **cobertura de controles**: el porcentaje del tiempo en que la telemetría de seguridad de la estación estuvo efectivamente operando.

Es una métrica de auditoría real. Un control desplegado en el 100% de los endpoints pero operando el 91% del tiempo deja una ventana ciega del 9%, y esa ventana no aparece en ningún reporte de despliegue. Medirla convierte una premisa —"el NOC y el SOC comparten infraestructura"— en un número reportable.

Nota de denominación: el servicio se llamaba antes de una forma que sugería medir disponibilidad del endpoint. Se renombró conservando el identificador interno, de modo que el histórico del indicador sobrevive al cambio (deuda #28). Evidencia del antes y el después en `evidence/bloque4e/`.

---

## 4. Triggers de medición dedicados

Los triggers que alimentan el indicador están en severidad `Warning` **de forma deliberada** — por debajo del umbral de la acción de notificación.

La severidad expresa acá **función y no criticidad**: el trigger alimenta el indicador sin generar notificación, separando el canal de medición del canal de alertado.

Fue necesario porque la etiqueta de disponibilidad de las plantillas oficiales está compartida por tres triggers de naturaleza distinta, y **la caída del agente de monitoreo no constituye indisponibilidad del servicio de negocio**, sino pérdida de la capacidad de observarlo. Confundir ambas cosas inflaría la indisponibilidad reportada con incidentes que el negocio nunca sufrió.

Etiquetado selectivo en origen: `sla: network-core` en el trigger de alcance del equipo, `sla: network-{#IFNAME}` en el prototipo de trigger de enlace.

---

## 5. Escalamiento

| Paso | Destino | Condición |
|---|---|---|
| 1 | Operador de primer nivel | — |
| 2 | Operador de primer nivel | Evento sin acuse de recibo |
| 3 | **Grupo** de segundo nivel | Evento sin acuse de recibo |

Intervalo de cinco minutos. **Derivado del presupuesto de error del objetivo, no de un valor arbitrario:** con un objetivo de 99.9% semanal sobre la ventana declarada, el margen es de unos nueve minutos por semana. Un escalamiento de treinta minutos sería decorativo — cuando el segundo nivel se enterara, el objetivo ya estaría incumplido varias veces.

El tercer paso apunta a un grupo y no a un usuario: un turno de segundo nivel es un rol con varias personas, y sumar un integrante no debe requerir modificar la acción.

El escalamiento cambia además el significado del acuse de recibo: deja de ser una marca de lectura y pasa a ser **asunción de responsabilidad**, porque detiene la cadena. Validado en las dos direcciones — la escalada completa y su interrupción por acuse. La segunda prueba es la que importa: un escalamiento que no se detiene cuando el incidente ya tiene dueño es peor que no tenerlo.

---

## 6. Ventanas de mantenimiento

Modalidad **con recolección activa**, con filtrado por etiquetas.

La modalidad sin recolección deja un hueco de datos indistinguible de un fallo de monitoreo en una investigación posterior. Desde la perspectiva de seguridad, **un período de mantención sin recolección es un período en el que la actividad no genera evidencia.**

**El filtro por etiquetas es la decisión de diseño del componente.** Una ventana que suprime todo suprime también las alertas de seguridad — y el mantenimiento programado es precisamente el momento en que la detención de servicios resulta esperada y no se cuestiona. Se configuró para que los problemas de disponibilidad y rendimiento se supriman y los de seguridad no.

Verificado: notificación de manipulación de controles de seguridad recibida con la ventana activa sobre el host.

---

## 7. Limitaciones declaradas del modelo

**El indicador en período calendario no es interpretable en este laboratorio.** El entorno no corre de forma continua; se enciende para trabajar. El denominador de cada semana depende de cuántas horas estuvo encendido, así que dos semanas consecutivas no son comparables. Por eso el reporte de disponibilidad es de **período declarado** y no mensual. Ver `reports/disponibilidad-2026-08-17.md`.

**La base de datos se mide a nivel TCP** — puerto abierto, no consulta real (deuda #11). Un motor que acepta conexiones y rechaza consultas aparecería disponible.

**Un servicio recién creado no evalúa un problema ya activo** al momento de su creación (deuda #30, hipótesis de latencia de sincronización, sin verificar).

**La profundidad efectiva del escalamiento tiene una anomalía sin resolver:** en el incidente espontáneo del 17 de agosto, el tercer paso no registró notificación. Hipótesis: permiso del grupo de segundo nivel sobre el grupo de hosts (deuda #31). Está declarado como hipótesis porque no se verificó, no como causa.
