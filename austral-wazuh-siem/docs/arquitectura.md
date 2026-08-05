# Arquitectura del laboratorio
**AustralPay es una organización ficticia de laboratorio.**

## Visión general

El mini-SOC se construye sobre **Wazuh** en despliegue *single-node* (manager + indexer + dashboard en un mismo host, vía Docker Compose). Los equipos monitoreados reportan al manager mediante el agente de Wazuh; la estación Windows suma **Sysmon** para telemetría de proceso enriquecida.

```
                        ┌─────────────────────────────┐
                        │   austral-siem (.50)         │
                        │   Wazuh single-node (Docker) │
                        │   manager + indexer + dashboard
                        └──────────────▲──────────────┘
                                       │  (agente 1514/1515)
        ┌──────────────────┬──────────┴───────────┬──────────────────┐
        │                  │                       │                  │
┌───────▼───────┐  ┌───────▼───────┐      ┌────────▼────────┐  ┌──────▼───────┐
│ austral-app   │  │ austral-db    │      │ austral-ws01    │  │  Kali (.100) │
│ .55 Ubuntu    │  │ .60 Ubuntu    │      │ .65 Windows 10  │  │  atacante    │
│ app de pagos  │  │ DB / atacante │      │ + Sysmon        │  │  (externo)   │
│ (objetivo)    │  │               │      │ (punto entrada) │  │              │
└───────────────┘  └───────────────┘      └─────────────────┘  └──────────────┘
```

## Componentes

| Componente | Función |
|---|---|
| **Wazuh manager** | Recibe eventos de los agentes, aplica el ruleset (base + custom), genera alertas, ejecuta Active Response |
| **Wazuh indexer** (OpenSearch) | Almacena e indexa alertas para búsqueda |
| **Wazuh dashboard** | Interfaz de análisis (búsqueda de eventos, MITRE, FIM, vulnerabilidades) |
| **Agente Wazuh** | Recolecta logs y eventos en cada host monitoreado |
| **Sysmon** (SwiftOnSecurity) | En Windows: telemetría de creación de procesos, registro, red, con linaje padre-hijo |

## Decisiones de diseño

- **Single-node**, no cluster: suficiente para un laboratorio; reduce consumo de RAM (OpenSearch es exigente) y complejidad. En producción se escalaría a multi-node.
- **Docker Compose**: despliegue reproducible y versionado, fácil de recrear.
- **Version pinning de agentes**: los agentes se fijan a la versión del manager (`4.9.2`) con `apt-mark hold` para evitar que el repo `stable` instale una versión más nueva y los desalinee.
- **Sysmon con config SwiftOnSecurity**: baseline comunitaria bien tuneada, en vez de configurar Sysmon desde cero.
- **Red bridged 192.168.1.0/24**: los hosts se ven entre sí como en una LAN corporativa, lo que habilita escenarios de movimiento lateral realistas.

Ver detalle de despliegue en [`despliegue.md`](./despliegue.md).
