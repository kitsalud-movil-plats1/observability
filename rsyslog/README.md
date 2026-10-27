# Logs centralizados con rsyslog en kit01

Eventos de DHCP, DNS, firewall y autenticación de todo el kit en un solo lugar (R12, D-09). rsyslog recibe en kit01 los logs de las VMs y de sw01 y los guarda por equipo en `/var/log/kitsalud/`, junto con una copia de los eventos de kit01. Así se puede diagnosticar desde kit01 aunque una VM esté caída. El AP no envía syslog y su registro se consulta en su página.

| Archivo | Qué es |
|---|---|
| [`../ansible/roles/rsyslog_central/`](../ansible/roles/rsyslog_central/) | Receptor en kit01 (`/etc/rsyslog.d/30-kitsalud-central.conf`), carpetas del log central y rotación (`/etc/logrotate.d/kitsalud-registros`). rsyslog 8.2312.0 con versión fija (`paquetes.yml`) y retenido |
| `platform/ansible/roles/comun/` | Reenvío de cada VM (`/etc/rsyslog.d/90-reenvio.conf`) y lectura de archivos con imfile (`comun_rsyslog_archivos`) |
| `platform/ansible/roles/bind9/` | El canal `default_syslog` de BIND9, que manda a syslog los eventos generales de `named` |
| `network/switch/sw01.rsc` | Acción `remote` de sw01 hacia `10.20.20.10:514` |

## Qué llega y dónde queda

| Origen | Cómo llega | Carpeta | Archivos |
|---|---|---|---|
| clinica01, comunidad01 | Todo su syslog por TCP 514 a `10.20.20.10` (F-15), con cola en disco. clinica01 agrega el `auth.log` de Samba con imfile (etiqueta `samba-auth`) | `clinica01/`, `comunidad01/` | `auth.log` (facilidades `auth` y `authpriv`, SSH, sudo y Samba) y `syslog.log` (lo demás) |
| sw01 | Temas `info`, `warning`, `error` y `critical` por UDP 514 a `10.20.20.10` (F-16), en formato BSD | `sw01/` | `syslog.log` (logins, cambios de configuración, enlaces) |
| kit01 | Copia de sus propios eventos. Los originales siguen en `/var/log/syslog`, `kern.log` y `auth.log` | `kit01/` | `firewall.log` (`fw-drop`), `dhcp.log` (Kea DHCPv4 y DHCPv6), `dns.log` (eventos generales de BIND9) y `auth.log` |
| Una IP que no está en la lista | Solo si nftables la deja pasar | `desconocido/` | `auth.log` y `syslog.log` |

**El equipo sale de la IP de origen**, no del nombre que trae el mensaje. La lista es `registros_equipos` (`platform/ansible/inventarios/kit/group_vars/all/registros.yml`), con las dos direcciones de cada equipo tomadas de `red.yml`. Así nadie puede escribir en la carpeta de otro, y no importa cómo cada equipo arma la cabecera.

**Las consultas DNS no se centralizan.** Son muchas y quedan en `/var/log/named/consultas.log` de kit01, con la rotación propia de BIND9.

**Escucha.** rsyslog escucha en 514/tcp y 514/udp solo en `10.20.20.10` y `fd5a:fc7e:d716:20::10`. nftables deja pasar el TCP solo desde las VMs por `br-srv` y el UDP solo desde sw01 por `lan0.10`, en las dos familias. Lo demás termina en `fw-drop`. Lo remoto entra a un ruleset propio, así que no se mezcla con el `/var/log/syslog` de kit01. rsyslog no hace consultas DNS inversas (`net.enableDNS="off"`), porque sin WAN cada consulta a los DNS del sitio quedaría esperando.

**Familias.** Las VMs y sw01 envían a la dirección IPv4 de servicios. El receptor atiende igual en IPv6, y así se probó desde las dos VMs y desde sw01.

**Rotación.** Semanal, con 4 copias comprimidas (un mes). Un archivo que pase de 100 MB rota antes, para que el `fw-drop` no llene el disco. Los archivos son `syslog:adm 0640`.

## Cómo se despliega

Desde `platform/ansible/`.

```bash
ansible-playbook playbooks/registros.yml          # receptor en kit01
ansible-playbook playbooks/dns.yml                # canal syslog de BIND9
ansible-playbook playbooks/comun.yml -l vms       # reenvío de las VMs
```

Antes hacen falta las reglas F-15 y F-16 de nftables (`network/kit01/nftables/`). sw01 se configura a mano por SSH a través de kit01 en Safe Mode, con las líneas de `/system logging` de `sw01.rsc`.

- **Validación.** Las dos plantillas de rsyslog se validan con `rsyslogd -N1` antes de copiarse. AppArmor solo deja a rsyslogd leer sus propias rutas, así que se valida una copia en `/etc/rsyslog.d/` con un nombre que no termina en `.conf`. La del reenvío lleva además el `workDirectory` de `rsyslog.conf`, que imfile necesita.
- **imfile y AppArmor.** imfile abre `/`, `/var/` y `/var/log/` para vigilar el archivo, y el perfil de rsyslogd solo permite lo que hay dentro de `/var/log/`. El rol `comun` agrega esas tres rutas a `/etc/apparmor.d/local/usr.sbin.rsyslogd`, solo en los equipos con `comun_rsyslog_archivos`, y recarga el perfil antes de reiniciar rsyslog.
- **Formato.** Las VMs reenvían con `RSYSLOG_ForwardFormat`, que conserva la hora con fracciones de segundo y la zona horaria.
- **Laboratorio.** `kitlab-cliente` es el único equipo de la lista (`inventarios/lab/group_vars/kit.yml`). Como no es una VM ni sw01, nftables no lo deja llegar al 514, y para probar el receptor desde él hay que abrirle el puerto con una regla temporal.

## Cómo se verifica

En kit01.

| Comando | Resultado esperado |
|---|---|
| `sudo rsyslogd -N1` | `End of config validation run` sin errores |
| `ss -lntup \| grep ':514 '` | TCP y UDP en `10.20.20.10` y `[fd5a:fc7e:d716:20::10]` |
| `sudo ls /var/log/kitsalud/` | `clinica01`, `comunidad01`, `kit01` y `sw01` |
| Un SSH con un usuario inexistente hacia clinica01 por `10.20.20.11` y por `fd5a:fc7e:d716:20::11`, y luego `sudo grep intruso /var/log/kitsalud/clinica01/auth.log` | `Invalid user intruso` desde `10.20.20.1` y desde `fd5a:fc7e:d716:20::10` |
| Un `kinit` con la clave mala en clinica01 (un solo intento, para no bloquear la cuenta) y luego `sudo grep samba-auth /var/log/kitsalud/clinica01/auth.log` | `NT_STATUS_WRONG_PASSWORD` con el usuario |
| `sudo tail /var/log/kitsalud/kit01/firewall.log` | Líneas `fw-drop` |
| Encender `prueba01` (`playbooks/prueba01.yml`) y `sudo grep DHCP4_LEASE_ALLOC /var/log/kitsalud/kit01/dhcp.log` | El préstamo de su MAC (`52:54:00:40:00:50`) |
| Entrar a sw01 y `sudo tail /var/log/kitsalud/sw01/syslog.log` | `user admin logged in from 10.20.10.1 via ssh` |
| `logger -T -n fd5a:fc7e:d716:20::10 -P 514 prueba` en una VM | La línea en `syslog.log` de esa VM |
| `sudo nft list chain inet filtro input \| grep 'dport 514'` | Contadores que suben en las cuatro reglas |

**Cola.** Con rsyslog detenido en kit01 (`sudo systemctl stop syslog.socket rsyslog`, porque si solo se detiene `rsyslog` el socket lo vuelve a levantar en el mismo segundo), lo que escriben las VMs queda en `/var/spool/rsyslog/` y llega con su hora original menos de 20 s después de `sudo systemctl start syslog.socket rsyslog`.

## Diagnóstico

- **No llega nada de una VM.** En la VM, `systemctl status rsyslog` y `journalctl -u rsyslog -n 20` (un `omfwd` suspendido dice que no pudo conectar). Desde la VM, `nc -zv 10.20.20.10 514`. Si no conecta, revisar en kit01 el contador de la regla del 514 y `ss -lnt | grep 514`.
- **No llega nada de sw01.** En sw01, `/system logging action print detail where name=remote` (`remote=10.20.20.10`, `bsd-syslog=yes`) y `/system logging print` (las cuatro reglas con `action=remote`). Un `:log info prueba` tiene que aparecer en `sw01/syslog.log`.
- **Los mensajes caen en `desconocido/`.** El equipo envía desde una dirección que no está en `registros_equipos` (por ejemplo, otra IPv6). Se agrega la dirección al inventario y se vuelve a aplicar `registros.yml`.
- **No llegan las líneas nuevas de Samba.** Revisar denegaciones de AppArmor en clinica01 con `journalctl -k | grep 'apparmor="DENIED".*rsyslogd'` y que `/etc/apparmor.d/local/usr.sbin.rsyslogd` tenga el bloque del rol `comun`.
- **rsyslog no arranca en kit01 después de un cambio.** `sudo rsyslogd -N1` muestra el error con el archivo y la línea. Si las direcciones de escucha no existen, `ip addr show br-srv` tiene que mostrar `.10` y `::10` (las mantiene `srv-dummy0`).
