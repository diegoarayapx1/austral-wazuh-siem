# Monitoreo SNMPv3 del equipo de borde

> AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.
>
> Direccionamiento en rango de documentación RFC 5737. Ver la nota del `README.md` de la raíz.

---

## 1. Por qué SNMPv3 y no v2c

SNMPv2c autentica con una cadena de comunidad que viaja **en texto plano** en cada consulta. Cualquiera con captura de tráfico en el segmento la obtiene, y con ella obtiene lectura completa del equipo: tabla de rutas, interfaces, vecinos, inventario.

SNMPv3 en nivel `authPriv` aporta autenticación por usuario y cifrado del contenido. En un laboratorio nadie está capturando el tráfico; se usa igual porque el objetivo es que el laboratorio se parezca a lo que se hace, no a lo que alcanza.

| Parámetro | Valor |
|---|---|
| Versión | SNMPv3 |
| Nivel de seguridad | `authPriv` |
| Protocolo de autenticación | SHA1 |
| Protocolo de privacidad | AES128 |
| Usuario | Referenciado por macro |

---

## 2. Defensa en profundidad: la ACL debajo del cifrado

El cifrado protege el contenido de la consulta. **No impide que alguien la haga.** Un atacante con las credenciales, o probando contra un equipo mal configurado, alcanza el agente igual.

Por eso el acceso SNMP está restringido por lista de control de acceso a la dirección del servidor de monitoreo, aplicada en el propio equipo:

```
ip access-list standard SNMP-ACCESS
 permit host 192.0.2.70
 deny   any log

snmp-server group  ... v3 priv access SNMP-ACCESS
snmp-server user   ... auth sha ... priv aes 128 ...
```

Son dos capas con funciones distintas: **la ACL previene, el cifrado protege lo que pasó la ACL.** El `deny any log` es el que convierte un intento fallido en evidencia — un contador que sube y una línea en el registro del equipo. Sin `log`, un barrido de reconocimiento contra el agente SNMP no deja huella.

Verificado desde un origen no autorizado: timeout del lado del cliente, contador de denegaciones incrementado y entrada en el registro del lado del equipo. Evidencia declarada en `evidence/README.md`.

---

## 3. Credenciales en macros de tipo secreto

Las tres credenciales están declaradas como macros a nivel de host, tipo `Secret text`:

```yaml
macros:
  - macro: '{$SNMPV3.AUTHPASS}'
    type: SECRET_TEXT
  - macro: '{$SNMPV3.PRIVPASS}'
    type: SECRET_TEXT
  - macro: '{$SNMPV3.USER}'
    type: SECRET_TEXT
```

Y referenciadas desde la interfaz:

```yaml
details:
  version: SNMPV3
  securityname: '{$SNMPV3.USER}'
  securitylevel: AUTHPRIV
  authprotocol: SHA1
  authpassphrase: '{$SNMPV3.AUTHPASS}'
  privprotocol: AES128
  privpassphrase: '{$SNMPV3.PRIVPASS}'
```

**Verificación:** el export de `templates/zbx_export_hosts.yaml` muestra las macros declaradas **sin campo `value`**. El valor no sale del sistema ni siquiera exportando la configuración, que es exactamente el comportamiento que justifica el tipo. Es una propiedad verificable contra un artefacto, no una afirmación.

La migración desde valores literales fue la deuda #19. El antes y el después están en `evidence/bloque4c/`, con las passphrases tachadas.

> **Deuda #32, abierta:** las passphrases estuvieron visibles en capturas de trabajo durante la ejecución del proyecto. El tachado protege el repositorio; corresponde rotarlas en el equipo y en las macros. Ver [`estado-y-deuda-tecnica.md`](estado-y-deuda-tecnica.md).

---

## 4. El identificador de motor SNMP

SNMPv3 deriva las claves de autenticación y privacidad combinando la passphrase con el **identificador de motor** del agente. Si el identificador cambia, las claves derivadas dejan de coincidir y la autenticación falla — aunque la passphrase sea exactamente la misma.

El equipo regenera ese identificador al arrancar. En un reinicio, el monitoreo empieza a reportar fallo de autenticación contra un host cuyas credenciales nadie tocó.

```
snmp-server engineID local <valor fijo>
```

Es el mismo tipo de problema que el índice de interfaces: **estado que el equipo regenera al arrancar y que el monitoreo asume estable.** Ninguno de los dos produce un error interpretable si no se sabe qué buscar. El síntoma —`Authentication failure (incorrect password, community or key)` sobre credenciales intactas— está en `evidence/bloque4a/22b-engineid-fallo-autenticacion.png`.

---

## 5. Descubrimiento de interfaces

La regla de descubrimiento del fabricante excluye por defecto las interfaces administrativamente caídas. Es un criterio sensato en un equipo de acceso con decenas de puertos libres y equivocado en un equipo de borde con dos interfaces relevantes: al apagar `Gi0/1` para simular la caída, **la interfaz dejó de cumplir el filtro, la plataforma la trató como recurso perdido y desactivó su trigger**.

El filtro se sobrescribió con una macro a nivel de host, no editando la plantilla:

```
{$NET.IF.IFADMINSTATUS.NOT_MATCHES} = ^$
```

La contrapartida se declara: el inventario pasa a incluir interfaces apagadas de forma permanente. **El filtro de descubrimiento es una decisión por tipo de equipo, no un valor global.**

---

## 6. Un resultado inesperado que vale la pena

Los ítems descubiertos incorporaron como etiqueta el texto del campo de descripción configurado en el propio equipo. La documentación escrita en la configuración del dispositivo **llegó sola hasta la consola de monitoreo**.

Es un argumento verificable de por qué se describen las interfaces, más allá de la buena práctica declarativa: el texto que uno escribe en el `description` de una interfaz termina siendo lo que el operador de guardia lee a las tres de la mañana.
