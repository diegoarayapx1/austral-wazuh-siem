# templates

## `zbx_export_hosts.yaml`

Export del host `austral-net` (Cisco vIOS por SNMPv3) y su grupo.

**Al importar:** las tres macros SNMPv3 llegan declaradas como `SECRET_TEXT` **sin valor**. Hay que completarlas a mano — el tipo de macro impide que el valor salga en un export, que es exactamente el comportamiento que se buscaba.

```yaml
{$SNMPV3.USER}
{$SNMPV3.AUTHPASS}
{$SNMPV3.PRIVPASS}
```

**Dependencia:** el export referencia la plantilla oficial `Cisco IOS by SNMP` por nombre, no la incluye. Tiene que estar presente en la instancia antes de importar.

La macro `{$NET.IF.IFADMINSTATUS.NOT_MATCHES}` viene sobrescrita a `^$` de forma deliberada, para que el descubrimiento no excluya interfaces administrativamente caídas. La justificación está en `../docs/snmp-config.md`.

## Lo que no está acá

**El dashboard del muro NOC.** Zabbix 7.0 no exporta dashboards globales desde el frontend — solo los de plantilla. No hay artefacto importable de ese componente.

Su registro es `../evidence/bloque4b/05-muro-noc.png` y la descripción de widgets en `../docs/arquitectura.md`. Reproducirlo requiere recrearlo a mano o consultarlo por API (`dashboard.get`).

## Direccionamiento

Este export conserva el direccionamiento privado real del laboratorio (`192.168.1.75`), a diferencia de los documentos de `../docs/`, que lo normalizan al rango RFC 5737.

Es deliberado: un export es un **artefacto funcional**, no prosa. Normalizarle la dirección produciría un archivo que importa limpio y no alcanza nada, con un error que no se manifiesta hasta el primer ciclo de sondeo. La dirección hay que cambiarla al importar en otro entorno de todos modos.
