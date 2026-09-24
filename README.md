# observability

Monitoreo y registros centralizados (mon01, `monitoreo.salud.movil`).

| Ruta | Contenido |
|---|---|
| `prometheus/` | Scrape configs (node, blackbox, snmp), reglas de alerta |
| `grafana/` | Provisioning de datasources y dashboards, login LDAP (Samba AD) |
| `loki/` | Configuración y retención de logs |
| `alloy/` | Agente de recolección (journal, syslog de OPNsense, switch y AP) |
