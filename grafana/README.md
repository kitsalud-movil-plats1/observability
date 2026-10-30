# Grafana en kit01

Vista del estado del kit para el personal y para quien lo administra (R12, D-09), con inicio de sesión centralizado en Samba AD (R10, P7). Grafana corre en kit01, así que sigue disponible aunque una VM esté caída, y lee Prometheus en el mismo equipo ([`../prometheus/`](../prometheus/)). Se entra por `https://monitoreo.salud.movil`.

| Archivo | Qué es |
|---|---|
| [`dashboards/estado.json`](dashboards/estado.json) | **Estado del kit**, la página de inicio |
| [`dashboards/recursos.json`](dashboards/recursos.json) | **Recursos de los equipos** |
| [`../ansible/roles/grafana/`](../ansible/roles/grafana/) | Grafana OSS 13.2.4 del repositorio `apt.grafana.com`, retenido, en `127.0.0.1:3000`, con `grafana.ini`, `ldap.toml`, el datasource de Prometheus y el sitio `monitoreo` de nginx |

El rol `nginx` de `platform/ansible` instala nginx 1.24 (retenido) y sin el sitio de fábrica. El sitio `monitoreo` escucha en el 443 de todas las direcciones con el certificado `monitoreo` que firma la raíz del kit (`/etc/kitsalud/tls/`). nftables solo deja llegar a ese puerto a las IP de administración y a NetBird (F-01), y el portal cautivo irá en su propia dirección (`10.20.40.1`).

## Dashboards

Están pensados para leerse de un vistazo. Verde funciona, rojo no, y cada panel dice en su descripción (el ícono `i`) qué mide.

**Estado del kit**

| Panel | Qué responde |
|---|---|
| Problemas ahora | Cuántas alertas hay activas. Lo normal es "Ninguno" |
| Equipos | Si kit01, clinica01, comunidad01, el switch y el Wi-Fi responden |
| Servicios | Si funcionan la historia clínica (DHIS2), el registro de pacientes, la consulta de prerregistros, la biblioteca, los videos, el portal de la Comunidad, los archivos compartidos, el inicio de sesión (LDAPS de AD), la base de datos, el DNS, el DHCP, la hora y el acceso de administración (SSH). Cada uno se prueba en IPv4 y en IPv6, y basta que falle uno de los dos para que salga en rojo |
| Qué está fallando | Las alertas activas en español, con el servicio, el equipo y si falla en IPv4 o en IPv6 |
| Si algo está en rojo | Los tres primeros pasos del diagnóstico rápido (sección 12 del documento) |

**Recursos de los equipos**

| Panel | Qué responde |
|---|---|
| Encendido desde hace | Tiempo desde el último arranque de cada equipo |
| Procesador en uso y Memoria en uso | Cómo están ahora, en verde, amarillo y rojo. La memoria de kit01 incluye la que usan las VMs |
| Disco ocupado | Cada disco de cada equipo. Rojo desde el 80 %, igual que la alerta `DiscoAlto` |
| Procesador y memoria en las últimas horas | Si un pico fue momentáneo o si algo va subiendo |

Los dashboards son de solo lectura en la interfaz (`allowUiUpdates: false`). Para cambiarlos se edita el JSON en este repositorio y se aplica el playbook. Grafana relee la carpeta cada 30 s.

## Inicio de sesión

| Cuenta | Cómo entra | Rol |
|---|---|---|
| Usuarios del grupo AD `admins` (por ejemplo `laura.gomez`) | LDAPS contra `dc01.ad.salud.movil:636` (F-13) con su propia clave, sin cuenta de servicio | Admin |
| Usuarios fuera de `admins` (por ejemplo `ana.rojas`) | Samba acepta la clave, pero Grafana no los deja entrar | Ninguno |
| `admin` local | Clave del vault (`vault_grafana_admin_password`) | Administrador de Grafana. Es la cuenta de respaldo cuando clinica01 no está |

Grafana verifica el certificado del DC con la raíz del kit (`/etc/grafana/ca-kit.crt`). Ese certificado lo firma el intermedio de clinica01 (rol `ca_interna`, `ca_hojas_vm`), porque el que Samba genera solo no trae subjectAltName y Go no lo acepta. kit01 resuelve con los DNS del sitio, así que el rol pone `dc01.ad.salud.movil` en `/etc/hosts` con las dos direcciones de clinica01.

## Cómo se despliega

Desde `platform/ansible/`, con la CA del kit (`playbooks/ca.yml`) y el certificado LDAPS de Samba (`playbooks/clinica01.yml` y después `playbooks/samba.yml`) ya aplicados.

```bash
ansible-playbook playbooks/monitoreo.yml
```

- **Primer arranque.** Grafana tarda unos 30 s en crear su base. El rol espera a `/api/health` antes de seguir.
- **Clave de admin.** `admin_password` de `grafana.ini` solo vale al crear la base, y `grafana.ini` la lee de `/etc/grafana/admin.clave` para que no aparezca en el `--diff`. Si alguien la cambia en la interfaz, la siguiente ejecución la devuelve a la del vault.
- **Sin Internet.** Sin reportes de uso, sin noticias, sin buscar actualizaciones y sin instalar plugins al arrancar (`preinstall_disabled`). La interfaz está en español y con la hora de Bogotá.

## Cómo se verifica

| Comando | Resultado esperado |
|---|---|
| En kit01, `curl --cacert /etc/prometheus/ca-kit.crt --resolve monitoreo.salud.movil:443:<10.20.20.10 o [fd5a:fc7e:d716:20::10]> https://monitoreo.salud.movil/api/health` | `200` con `"database": "ok"` en las dos familias |
| Lo mismo desde una laptop por NetBird, con `100.90.225.113` | `200` |
| Desde sw01 (`10.20.10.2`, Interna sin ser IP admin), `/tool fetch url=https://10.20.20.10/` | Sin respuesta y `fw-drop` con `DPT=443` en `journalctl -k` de kit01 |
| `POST /login` con `laura.gomez`, `ana.rojas` y `admin` | `200` con rol Admin, `401` y `200` |
| `grep "simple bind/TLS" /var/log/samba/auth.log` en clinica01 | Los intentos desde `10.20.20.1` |
| Con clinica01 apagada, `POST /login` con `admin` | `200`, y el dashboard muestra clinica01 y sus servicios en rojo |

**Desde una laptop.** Con NetBird activo, se agrega `100.90.225.113 monitoreo.salud.movil` a `/etc/hosts` y se abre `https://monitoreo.salud.movil`. Para no ver la advertencia del navegador se instala la raíz del kit (`http://ca.salud.movil` desde la red del kit).

## Diagnóstico

- **`502 Bad Gateway`.** nginx responde pero Grafana no. `systemctl status grafana-server` y `journalctl -u grafana-server -n 50`.
- **No responde en el 443.** `ss -ltn | grep :443` y `nginx -t`. Si una ejecución del playbook se cortó después de activar el sitio, el aviso de recarga se pierde y nginx queda con la configuración anterior. En ese caso, `sudo nginx -t && sudo systemctl reload nginx`.
- **Un usuario de `admins` no puede entrar.** Desde kit01, `openssl s_client -connect dc01.ad.salud.movil:636 -verify_hostname dc01.ad.salud.movil -CAfile /etc/grafana/ca-kit.crt` tiene que terminar en `Verify return code: 0`. Si falla, se revisa que clinica01 esté encendida y que ufw tenga el 636 para kit01 (`sudo ufw status | grep 636`). Tras 5 claves equivocadas, AD bloquea la cuenta 15 minutos.
- **Los paneles dicen "Sin datos".** Prometheus no responde (`systemctl status prometheus`) o el equipo está apagado. Para un equipo apagado, los paneles de estado muestran "No responde".
