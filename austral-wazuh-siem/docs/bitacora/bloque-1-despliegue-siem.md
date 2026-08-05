# Bloque 1 — Fundamentos + instalación (Wazuh SIEM)
**AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio.**

Proyecto: Wazuh SIEM · AustralPay Lab
Sesiones: 1A, 1B

---

## Objetivo

Desplegar la base del proyecto de detección: servidor `austral-siem` operativo y accesible por red, con el stack Wazuh (Manager, Indexer, Dashboard) corriendo en modo single-node sobre Docker.

## Entorno

| Componente | Detalle |
|---|---|
| Hipervisor | VMware Workstation |
| SO | Ubuntu Server 24.04.4 LTS |
| Hostname | `austral-siem` |
| Recursos | 4 vCPU, 8 GB RAM, 50 GB disco |
| Red | Estática, `192.168.1.50/24`, red bridged |
| Orquestación | Docker Engine (instalación manual desde repositorio oficial) |
| Stack | Wazuh v4.9.2 — Manager, Indexer (OpenSearch), Dashboard |
| Modo | Single-node |

## Qué se hizo

1. Instalación de Ubuntu Server 24.04 LTS sobre VM dedicada, con IP estática configurada desde el instalador y particionado LVM sobre el disco completo.
2. Habilitación de acceso SSH con usuario dedicado; verificación de conectividad desde estación de trabajo externa.
3. Instalación de Docker Engine desde el repositorio oficial de Docker (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin`).
4. Ajuste de parámetro de kernel `vm.max_map_count=262144`, requerido por OpenSearch para el manejo de sus índices, persistido en `/etc/sysctl.conf`.
5. Clonación del repositorio oficial `wazuh-docker` en la versión estable `v4.9.2`.
6. Generación de certificados TLS internos entre los tres componentes del stack mediante el generador oficial de Wazuh.
7. Despliegue del stack completo con `docker compose up -d`.
8. Verificación de acceso al Dashboard vía HTTPS y confirmación de estado operativo de los tres servicios.

## Decisiones de diseño

- **Docker en vez de instalación nativa**: permite aislar cada componente del stack (Manager, Indexer, Dashboard) con sus dependencias propias y reconstruir el entorno de forma reproducible.
- **Instalación manual de Docker (repositorio oficial, no snap)**: da control de versión exacto y evita restricciones de sandboxing que pueden interferir con el manejo de volúmenes.
- **Despliegue single-node**: adecuado para entorno de laboratorio; en un despliegue productivo, el Indexer (OpenSearch) se implementaría como cluster multi-nodo con roles de manager e indexer separados, para soportar sharding, replicación y mayor volumen de eventos.
- **Versión fija del repositorio (`v4.9.2`)** en lugar de la rama de desarrollo, para asegurar un despliegue reproducible y documentado.

## Evidencia

- `evidence/01-wazuh-dashboard.png` — Dashboard de Wazuh accesible y operativo, mostrando el resumen de agentes y las alertas de las últimas 24 horas por nivel de severidad.

![Wazuh Dashboard activo](evidence/01-wazuh-dashboard.png)

## Estado del checklist

- [x] Servidor `austral-siem` desplegado y accesible
- [x] Docker instalado
- [x] Parámetros de kernel de OpenSearch configurados
- [x] Stack Wazuh single-node desplegado (Manager + Indexer + Dashboard)
- [x] Dashboard verificado y accesible

## Próximo paso

Bloque 2 — despliegue de agentes Wazuh en los hosts monitoreados (`austral-app`, `austral-db`) e integración de Sysmon en el endpoint Windows.

---
*AustralPay es una organización ficticia creada exclusivamente con fines de laboratorio y portafolio. Ninguna infraestructura, dato o incidente descrito corresponde a una empresa real.*
