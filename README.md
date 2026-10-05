# observability

Monitoreo y registros centralizados, en el host kit01 (`monitoreo.salud.movil`). Así se puede diagnosticar aunque una VM esté caída.

| Ruta | Contenido |
|---|---|
| `prometheus/` | Scrape configs (node_exporter de kit01 y de las VMs; sondas blackbox de DNS, HTTPS y puertos), reglas de alerta (se revisan en Grafana; sin Alertmanager) |
| `grafana/` | Provisioning de datasources y dashboards, login LDAPS contra Samba AD (con cuenta local de respaldo) |
| `rsyslog/` | Receptor central en kit01 (514/tcp desde las VMs, 514/udp desde switch y AP) y configuración de reenvío para los clientes |
