# observability

Monitoreo y registros centralizados, en el host kit01 (`monitoreo.salud.movil`). Así se puede diagnosticar aunque una VM esté caída.

| Ruta | Contenido |
|---|---|
| [`prometheus/`](prometheus/) | Reglas de alerta (se revisan en Prometheus y Grafana; sin Alertmanager) y la documentación de Prometheus, de las sondas blackbox de DNS, web, TCP y ping y del node_exporter de kit01 y las VMs |
| `grafana/` | Provisioning de datasources y dashboards, login LDAPS contra Samba AD (con cuenta local de respaldo) |
| [`rsyslog/`](rsyslog/) | Documentación de los logs centralizados en kit01 (514/tcp desde las VMs, 514/udp desde sw01; el AP no envía syslog), por equipo en `/var/log/kitsalud/` |
| `ansible/roles/` | Roles `prometheus`, `blackbox_exporter`, `node_exporter`, `grafana` y `rsyslog_central`. `platform/ansible` los encuentra por `roles_path` y los aplica con `playbooks/monitoreo.yml` y `playbooks/registros.yml` |
