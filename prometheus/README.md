# Prometheus y exporters en kit01

Estado de servicios, puertos, CPU, memoria y disco del kit (R12, D-09). Prometheus vive en kit01 para seguir viendo el kit aunque caiga una VM. Lee el `node_exporter` de kit01 y de las VMs y hace sondas blackbox de DNS, web, TCP y ping, en IPv4 y en IPv6 salvo el AP, que solo tiene IPv4. No hay Alertmanager, porque sin Internet no hay a dónde enviar las alertas. Se revisan en Prometheus y, con observability#2, en Grafana (`monitoreo.salud.movil`).

| Archivo | Qué es |
|---|---|
| [`alertas.yml`](alertas.yml) | Reglas `ServicioCaido`, `UnidadFallida`, `ExporterCaido` y `DiscoAlto`. El rol `prometheus` las copia a `/etc/prometheus/reglas/` y las valida con `promtool` |
| [`../ansible/roles/prometheus/`](../ansible/roles/prometheus/) | Prometheus 2.45.3 en `127.0.0.1:9090`, cada 15 s y con 30 días de retención. Los trabajos salen del inventario (`prometheus.yml.j2`) |
| [`../ansible/roles/blackbox_exporter/`](../ansible/roles/blackbox_exporter/) | blackbox 0.24.0 en `127.0.0.1:9115`, con un módulo por consulta DNS y por servicio web (`blackbox.yml.j2`) |
| [`../ansible/roles/node_exporter/`](../ansible/roles/node_exporter/) | node_exporter 1.7.0 con el colector systemd de las unidades críticas. En kit01 escucha en `127.0.0.1:9100` y en las VMs en el 9100, con ufw abierto solo para kit01 |

Los tres son paquetes de Ubuntu 24.04 con versión fija (`paquetes.yml`) y retenidos con `apt-mark hold`. Ninguno escucha fuera de localhost en kit01, así que nftables no cambia.

## Qué se mide

| Trabajo | Destinos | Qué comprueba |
|---|---|---|
| `node` | kit01, clinica01 y comunidad01 | CPU, memoria, disco, hora (`node_timex_sync_status`) y estado de `named`, `kea-dhcp4-server`, `kea-dhcp6-server`, `radvd`, `chrony`, `nftables`, `netbird`, `libvirtd`, `samba-ad-dc` y `docker` |
| `sonda_dns` | BIND9 en `10.20.20.10` y `fd5a:fc7e:d716:20::10` | A y AAAA de `pacientes.salud.movil` y A de `dc01.ad.salud.movil` (delegación a Samba AD). La respuesta tiene que traer la dirección esperada |
| `sonda_web` | `pacientes` (`/api/ping`) y `registro` por HTTPS, `biblioteca` y `ca` por HTTP | Respuesta `2xx`. En HTTPS además valida el certificado con la raíz de la CA del kit |
| `sonda_tcp` | clinica01 en el 445 (SMB) y el 5432 (PostgreSQL) | El puerto acepta conexiones |
| `sonda_icmp` | sw01, ap01 (solo IPv4), clinica01 y comunidad01 | Eco ICMP e ICMPv6 |
| `prometheus` y `blackbox` | kit01 | Los propios servicios de monitoreo |

Las listas están en `platform/ansible/inventarios/kit/group_vars/all/monitoreo.yml` y las direcciones salen de `red.yml`. Cada sonda lleva las etiquetas `servicio`, `equipo` y `familia` (`ipv4` o `ipv6`).

**Sondas web por IP.** kit01 resuelve con los DNS del sitio, así que las sondas web y TCP van a la IP de cada familia y el módulo pone el nombre en la cabecera `Host` y en el SNI. Así cada sonda prueba una familia concreta, y el nombre lo comprueban por separado las sondas DNS contra BIND9. Las sondas no siguen redirecciones, porque la redirección lleva al nombre completo. Por eso `pacientes` se prueba en `/api/ping`, que responde 200 sin iniciar sesión.

**Node por IPv4.** Prometheus lee el node_exporter de cada VM por IPv4 para no duplicar las series. El IPv6 de cada VM lo cubren las sondas de ping, web y TCP.

**Flujos.** kit01 llega a las VMs por F-14. ufw de cada VM abre a `10.20.20.1`, `10.20.20.10`, `fd5a:fc7e:d716:20::1` y `fd5a:fc7e:d716:20::10` el 9100 y los puertos de las sondas (`monitoreo_puertos` en `host_vars`), que son 443 y 445 en clinica01 y 80 y 443 en comunidad01. El 5432 lo publica Docker sin pasar por ufw y lo sigue filtrando `pg_hba.conf`, que rechaza la sesión de kit01 después de aceptar la conexión TCP.

## Alertas

| Alerta | Condición | Tiempo |
|---|---|---|
| `ServicioCaido` | Una sonda en `probe_success 0` | 30 s |
| `UnidadFallida` | Una unidad crítica en estado `failed` | 30 s |
| `ExporterCaido` | `up == 0` (equipo apagado o exportador caído) | 1 min |
| `DiscoAlto` | Un sistema de archivos `ext4`, `xfs` o `vfat` por encima del 80 % | 5 min |

El uso de disco se calcula con el espacio disponible para usuarios sin privilegios (`node_filesystem_avail_bytes`), así que da unos puntos más que `df`, que no cuenta el 5 % reservado para root.

## Cómo se despliega

Desde `platform/ansible/`.

```bash
ansible-playbook playbooks/monitoreo.yml                  # kit real
ansible-playbook -i inventarios/lab playbooks/monitoreo.yml   # laboratorio virtual
```

- **Orden.** Necesita la CA del kit (`playbooks/ca.yml`), porque el rol `blackbox_exporter` copia la raíz a `/etc/prometheus/ca-kit.crt`. `/etc/kitsalud` solo lo lee root y el exportador corre como `prometheus`.
- **Instalación.** Cada paquete se instala sin arrancar (`policy-rc.d`) y arranca ya con la configuración del kit, porque con la de fábrica escucharía en todas las direcciones.
- **ICMP.** Las sondas de ping necesitan `CAP_NET_RAW`. El paquete del blackbox solo se la da al binario si se responde que sí a `prometheus-blackbox-exporter/want_cap_net_raw` en debconf, y el rol lo hace antes de instalar.
- **Cambios.** Los cambios en la configuración o en las reglas recargan Prometheus (`systemctl reload`) sin perder lo que tiene en memoria. Los cambios en los argumentos (`/etc/default/prometheus`) lo reinician.
- **Laboratorio.** La única "VM" es `kitlab-cliente`, en la Interna (`inventarios/lab/group_vars/`). Las sondas del kit no tienen a quién llegar, así que todas quedan en 0 y disparan `ServicioCaido`.

## Cómo se verifica

En kit01.

| Comando | Resultado esperado |
|---|---|
| `ss -ltn \| grep -E ':(9090\|9100\|9115) '` | Los tres solo en `127.0.0.1` |
| `curl -s 127.0.0.1:9090/api/v1/targets \| grep -o '"health":"[a-z]*"' \| sort \| uniq -c` | 30 en `up` |
| `curl -s '127.0.0.1:9090/api/v1/query?query=probe_success==0'` | Resultado vacío |
| `curl -s 127.0.0.1:9090/api/v1/alerts` | Sin alertas |
| `curl -s '127.0.0.1:9115/probe?module=web_pacientes&target=https://10.20.20.12/' \| grep probe_success` | `0`, porque el intermedio de comunidad01 no puede firmar `pacientes` |
| `sudo systemctl stop named`, esperar y luego `sudo systemctl start named` | `ServicioCaido` de las seis sondas DNS. La primera pasó a `firing` a los 35 s y se resolvieron 30 s después de volver a arrancar `named` |
| `getcap /usr/bin/prometheus-blackbox-exporter` | `cap_net_raw=ep` |

En las VMs, `sudo ufw status | grep monitoreo` muestra las reglas para las cuatro direcciones de kit01.

**Ver la interfaz.** Prometheus no se publica en la red. Desde una IP de administración o NetBird, `ssh -L 9090:127.0.0.1:9090 kitsalud@<kit01>` y luego `http://localhost:9090/targets` y `http://localhost:9090/alerts`.

## Diagnóstico

- **Un target `down` en `node`.** La VM está apagada (`virsh list --all`) o el exportador no corre (`systemctl status prometheus-node-exporter` en la VM). Si responde en la VM pero no desde kit01, falta la regla de ufw (`sudo ufw status | grep 9100`).
- **Una sonda en 0.** `curl -s '127.0.0.1:9115/probe?module=<módulo>&target=<destino>&debug=true'` muestra cada paso de la sonda y el error. El módulo y el destino están en `/targets`, en las etiquetas `__param_module` e `instance`.
- **Las sondas ICMP fallan todas** con `socket: permission denied` en el `debug`. El binario perdió `CAP_NET_RAW`. Se vuelve a aplicar el playbook, que reconfigura el paquete.
- **Una sonda HTTPS falla con `x509`.** El certificado no es de la CA del kit o cambió la raíz. Se compara `/etc/prometheus/ca-kit.crt` con `/etc/kitsalud/ca/raiz.crt` y se aplica de nuevo el playbook.
- **Prometheus no arranca después de un cambio.** `journalctl -u prometheus -n 30` y `promtool check config /etc/prometheus/prometheus.yml`. El rol valida antes de copiar, así que suele ser un cambio hecho a mano.
