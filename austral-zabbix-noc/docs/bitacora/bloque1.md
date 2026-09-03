# Bloque 1 — Fundamentos + instalación + hosts (Zabbix NOC)
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Zabbix NOC · AustralPay Lab
Sesiones: 1A, 1B

---

## Objetivo

Desplegar la base del proyecto de monitoreo: servidor `austral-noc` operativo con el stack Zabbix (Server, Base de datos, Frontend) corriendo sobre Docker Compose, con los tres hosts de la infraestructura AustralPay agregados y reportando mediante agente en modo activo.

## Entorno

| Componente | Detalle |
|---|---|
| Hipervisor | VMware Workstation |
| SO | Ubuntu Server 24.04 LTS |
| Hostname | `austral-noc` |
| Recursos | 2 vCPU, 4 GB RAM, 30 GB disco |
| Red | Estática, `192.0.2.70/24`, red bridged |
| Orquestación | Docker Engine + Compose (instalación manual desde repositorio oficial) |
| Stack | Zabbix 7.0.29 LTS — Server, PostgreSQL 16.6, Frontend (nginx) |
| Hosts monitoreados | `austral-app`, `austral-db` (Linux), `austral-ws01` (Windows) |
| Modo de monitoreo | Agente activo |

## Qué se hizo

1. Instalación de Ubuntu Server 24.04 LTS sobre VM dedicada, con IP estática configurada por Netplan.
2. Instalación de Docker Engine y Docker Compose desde el repositorio oficial de Docker.
3. Despliegue del stack Zabbix (Server + PostgreSQL + Frontend) mediante `docker-compose.yml`, con credenciales gestionadas en archivo de variables de entorno separado.
4. Verificación de acceso al frontend vía navegador y rotación inmediata de la credencial administrativa por defecto.
5. Instalación del agente Zabbix (versión fijada, coincidente con la del server) en los tres hosts de la infraestructura, con `apt-mark hold` en los hosts Linux para prevenir desalineación de versión.
6. Configuración de los tres agentes en modo activo, apuntando al server.
7. Alta de los tres hosts en el frontend, con las plantillas oficiales correspondientes (Linux / Windows, variante activa).
8. Verificación de los tres hosts reportando datos en tiempo real.

## Decisiones de diseño

- **VM dedicada para el server de monitoreo**: aísla el consumo de recursos del stack de detección (SIEM) del stack de monitoreo (NOC), manteniendo una separación de roles clara entre ambos.
- **Docker Compose**: despliegue reproducible con versión de cada componente declarada explícitamente, sin depender de instalación manual paquete por paquete.
- **Versión fija (`7.0.29` LTS)**, no `latest`: reproducibilidad y trazabilidad de versión entre el server y los agentes.
- **Credenciales en archivo de entorno separado del compose**: permite publicar la configuración de despliegue sin exponer secretos.
- **Monitoreo activo** en los tres hosts: modo preferido en despliegues a escala (reduce carga en el server, tolera hosts detrás de NAT/firewall), aplicado de forma consistente desde el inicio.

## Evidencia

- `evidence/01-hosts-activos.png` — Los tres hosts de la infraestructura AustralPay (`austral-app`, `austral-db`, `austral-ws01`) en estado activo y reportando datos en tiempo real dentro del frontend de Zabbix.

![Hosts activos en Zabbix](evidence/01-hosts-activos.png)

## Estado del checklist

- [x] Servidor `austral-noc` desplegado y accesible
- [x] Docker instalado
- [x] Stack Zabbix desplegado (Server + BD + Frontend)
- [x] Tres hosts agregados con agente en modo activo
- [x] Hosts verificados reportando datos

## Próximo paso

Bloque 2 — monitoreo SNMP del dispositivo de red de borde (`austral-net`), configuración de triggers por severidad y notificaciones.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
