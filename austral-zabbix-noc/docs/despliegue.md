# Despliegue

> AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.
>
> Direccionamiento en rango de documentación RFC 5737. Ver la nota del `README.md` de la raíz.

---

## 1. Servidor de monitoreo

| Componente | Detalle |
|---|---|
| Hipervisor | VMware Workstation, red en modo puente |
| SO | Ubuntu Server 24.04 LTS |
| Recursos | 2 vCPU · 4 GB RAM · 30 GB disco |
| Red | Estática por Netplan, `192.0.2.70/24` |
| Orquestación | Docker Engine + Compose, desde el repositorio oficial |
| Stack | Zabbix 7.0.29 LTS — Server, PostgreSQL 16, Frontend nginx |

**VM dedicada.** Aísla el consumo de recursos del stack de monitoreo del stack de detección que corre en paralelo en el mismo laboratorio, y mantiene separación de roles entre ambos.

**Versión fija `7.0.29` LTS, no `latest`.** Reproducibilidad y trazabilidad de versión entre servidor y agentes. Un `latest` en un compose convierte el despliegue en algo que funciona hoy y no se sabe si funciona mañana.

**Credenciales en archivo de entorno separado del compose.** Permite publicar la configuración de despliegue sin exponer secretos. La credencial administrativa por defecto se rotó inmediatamente después del primer acceso al frontend.

---

## 2. Agentes

Agente Zabbix con **versión fijada, coincidente con la del servidor**, en los tres hosts. En los hosts Linux se aplicó `apt-mark hold` para prevenir que una actualización del sistema desalinee la versión.

Los tres configurados en **modo activo**, apuntando al servidor.

**Sobre la instalación silenciosa en Windows:** `msiexec /q` falla sin elevación y **no devuelve error**. Termina, no instala nada, y no dice nada. Es el primer caso del patrón que se repite en todo el proyecto: el fallo silencioso.

---

## 3. Conectividad

El sondeo ICMP hacia el host Windows requirió una regla de firewall explícita: Windows bloquea el eco entrante por defecto y el resultado es indistinguible de un host caído.

La regla se creó con **alcance mínimo**: solo eco ICMP entrante, y solo desde la dirección del servidor de monitoreo. No "permitir ICMP".

```powershell
New-NetFirewallRule -DisplayName "ICMPv4 Echo Request - Monitoreo AustralPay" `
  -Protocol ICMPv4 -IcmpType 8 -Direction Inbound `
  -RemoteAddress 192.0.2.70 -Action Allow
```

Evidencia del antes y el después en `evidence/bloque2/01-icmp-firewall-antes-despues.png`.

---

## 4. Reproducir esto

1. Ubuntu Server 24.04 LTS con IP estática.
2. Docker Engine y Compose desde el repositorio oficial de Docker.
3. Compose del stack Zabbix con versión fijada y variables de entorno en archivo separado.
4. Acceder al frontend y **rotar la credencial administrativa por defecto antes de cualquier otra cosa**.
5. Agente en cada host, misma versión que el servidor, modo activo.
6. Alta de hosts con las plantillas oficiales correspondientes.
7. Para el equipo de red: importar `templates/zbx_export_hosts.yaml` y completar las tres macros SNMPv3, que llegan declaradas y vacías. Ver [`snmp-config.md`](snmp-config.md).

El `docker-compose.yml` no se publica: contiene la estructura de credenciales del entorno. El stack es el oficial de Zabbix para PostgreSQL, sin modificaciones.

---

## 5. Limitaciones del despliegue

- **Sin alta disponibilidad y sin respaldo automatizado.** Es un laboratorio de una sola instancia. Un fallo del servidor de monitoreo es pérdida total del histórico.
- **Sin cifrado entre agente y servidor.** Zabbix soporta PSK y certificados; no se implementó. En una red de producción sería requisito, no opción.
- **Frontend sobre HTTP.** Mismo criterio.
- **El puerto de la base de datos quedó accesible a toda la red local** (deuda #12, pendiente).

Se declaran porque un despliegue de laboratorio presentado como si fuera de producción es peor que uno de laboratorio declarado.
