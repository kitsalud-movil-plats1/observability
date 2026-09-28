# observability

Monitoreo y registros centralizados (mon01, `monitoreo.salud.movil`).

| Ruta | Contenido |
|---|---|
| `prometheus/` | Scrape configs (node, blackbox), reglas de alerta (se revisan en Grafana; sin Alertmanager) |
| `grafana/` | Provisioning de datasources y dashboards, login LDAP (Samba AD) |
| `loki/` | Configuración y retención de logs |
| `alloy/` | Agente de recolección (journal, syslog de OPNsense, switch y AP) |
