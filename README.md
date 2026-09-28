# observability

Monitoreo y registros centralizados (rol mon01, `monitoreo.salud.movil`, en la VM ops01).

| Ruta | Contenido |
|---|---|
| `prometheus/` | Scrape configs (node, blackbox), reglas de alerta (se revisan en Grafana; sin Alertmanager) |
| `grafana/` | Provisioning de datasources y dashboards, login LDAP (Samba AD) |
| `rsyslog/` | Receptor central en mon01 (514/tcp desde VMs y nodos, 514/udp desde OPNsense, switch y AP) y configuración de reenvío para los clientes |

Loki queda como evolución (sección 12.1 de `docs/arquitectura/00-punto-de-partida.md`).
