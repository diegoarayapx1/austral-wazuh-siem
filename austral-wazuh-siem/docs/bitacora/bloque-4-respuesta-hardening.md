# Informe Técnico — Proyecto 1 (Wazuh SIEM) · Bloque 4: Active Response + Vulnerability Detection + FIM + Retención
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.**

## Objetivo
Implementar respuesta automática ante brute force, detección de vulnerabilidades por inventario de paquetes, monitoreo de integridad de archivos con atribución de usuario, y retención de logs alineada a PCI-DSS Requirement 10.

## Entorno
- `austral-siem` (192.168.1.50) — Wazuh manager, Docker Compose single-node, v4.9.2
- `austral-app` (192.168.1.55) — Ubuntu 24.04 LTS, agente Wazuh
- Atacante — Kali Linux (192.168.1.100), fuera del alcance de monitoreo

---

## 4A — Active Response

### Configuración
Dos triggers de `firewall-drop`, graduados por confianza de la señal:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763,5551,2502</rules_id>
  <timeout>180</timeout>
</active-response>

<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100220,100230</rules_id>
  <timeout>600</timeout>
</active-response>
```

Whitelist en `<global>` protege gateway, manager, hosts monitoreados y estación de administración de bloqueo accidental.

### Validación
Ataque de brute force SSH ejecutado contra `austral-app`. Ambos triggers dispararon correctamente:
- Trigger de bajo timeout (180s): bloqueo confirmado, reversión automática verificada con precisión de 1 segundo (181s medidos).
- Trigger de alto timeout (600s): disparado por regla custom de brute force exitoso (T1110.001/T1078), bloqueo confirmado en el pipeline completo (detección → active-response → iptables) en ~2 segundos.

### Decisión de diseño documentada
El active-response se limitó deliberadamente a `rules_id` explícitos de alta confianza, nunca a `level` genérico — evita que una regla nueva futura se convierta en trigger de bloqueo sin decisión consciente. Se documentó el riesgo de auto-DoS (IP spoofeada, NAT compartido, ataque desde infraestructura propia) como limitación conocida de cualquier active-response basado en IP de origen.

---

## 4B — Vulnerability Detection + FIM

### Vulnerability Detection
Módulo nativo de Wazuh (syscollector + feed de CVEs). Resultado sobre `austral-app`: 1,001 vulnerabilidades detectadas (190 High, 537 Medium, 9 Low, 0 Critical), incluyendo hallazgos en dependencias de aplicación (librería Python Twisted) además de paquetes del sistema operativo.

### FIM (File Integrity Monitoring)
Configurado en modo `whodata` sobre un directorio de aplicación crítico, además de los directorios de sistema ya cubiertos desde el despliegue inicial:

```xml
<directories whodata="yes" realtime="yes" report_changes="yes">/opt/austral-app/config</directories>
```

`whodata` requiere `auditd` como fuente de atribución de usuario/proceso — elegido sobre `realtime` simple por el requisito de trazabilidad de PCI-DSS Req. 11.5.

### Validación
Modificación real de un archivo de configuración simulando actividad post-compromiso. La alerta generada incluyó: hash antes/después (md5/sha1/sha256), y — vía `auditd` — el usuario, comando completo y TTY que ejecutó el cambio.

---

## 4C — Retención de logs (PCI-DSS Req. 10)

### Requisito
PCI-DSS Req. 10 exige retención mínima de 12 meses de logs de auditoría, con al menos 3 meses inmediatamente accesibles sin proceso de restauración.

### Configuración aplicada

**Logs de alertas** (gestionados nativamente por Wazuh vía `monitord`):
```
monitord.keep_log_days=365
```
(en `local_internal_options.conf`, corrigiendo el default de 31 días)

**Log de active-response** (sin rotación nativa, gestionado con `logrotate` del sistema operativo):
```
/var/ossec/logs/active-responses.log {
    weekly
    rotate 52
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    su wazuh wazuh
}
```

### Decisión de alcance
Se priorizó la retención de `alerts.log` y `active-responses.log` (evidencia de detección y de respuesta) sobre el registro completo de tráfico (`archives.log`/`logall`). En un entorno productivo real sujeto a PCI-DSS Req. 10.2, el registro completo sería obligatorio; en este laboratorio se documentó la decisión de alcance explícitamente como una limitación consciente del entorno de prueba, no como un descuido.

### Validación end-to-end
Se forzó una rotación real y se confirmó que el histórico se preservó correctamente (archivo comprimido `.1` con el contenido previo intacto). Se disparó un segundo active-response real tras la rotación para confirmar que el proceso de Wazuh continúa escribiendo sin pérdida de datos ni necesidad de reinicio — validando que `copytruncate` es compatible con el patrón de escritura del agente.

---

## Resumen de reglas y configuración

| Componente | Configuración | Estado |
|---|---|---|
| Active Response — brute force en curso | `rules_id=5763,5551,2502`, timeout 180s | Validado en pipeline real |
| Active Response — brute force exitoso | `rules_id=100220,100230`, timeout 600s | Validado en pipeline real |
| Vulnerability Detection | Feed nativo + syscollector | 1,001 CVEs detectadas, datos reales |
| FIM whodata | Directorio de aplicación crítico | Cambio detectado con atribución de usuario |
| Retención alerts | `monitord.keep_log_days=365` | 12 meses, conforme a PCI-DSS Req. 10 |
| Retención active-response | `logrotate`, rotación semanal, 52 rotaciones | 12 meses, validado con escritura post-rotación |

## Cierre del bloque
Los cuatro componentes del bloque quedan integrados en una cadena de valor completa: detección → respuesta automática → identificación de superficie de ataque no explotada → detección de actividad post-compromiso → evidencia retenida conforme a requisitos de cumplimiento. Cada componente fue validado con datos reales generados en el propio laboratorio, no con datos de ejemplo.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.*
