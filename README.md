# 🏠 Homelab — Nginx Reverse Proxy + SELinux + Firewalld

Guía de referencia y diagnóstico para montar un reverse proxy con Nginx en sistemas
con SELinux (RHEL, CentOS, Fedora, Rocky, AlmaLinux) y Firewalld.

> **Contexto:** homelab personal. Backend Python (Flask/FastAPI/Django) servido con
> gunicorn o uvicorn, detrás de Nginx como proxy inverso, con SELinux en Enforcing
> y Firewalld activo.

---

## 📋 Tabla de contenidos

1. [Arquitectura típica](#-arquitectura-típica)
2. [Conceptos clave](#-conceptos-clave)
3. [Diagnóstico rápido (orden recomendado)](#-diagnóstico-rápido-orden-recomendado)
4. [Errores comunes y su causa](#-errores-comunes-y-su-causa)
5. [Carpetas web fuera de las de defecto](#-carpetas-web-fuera-de-las-de-defecto)
6. [Reverse proxy y booleanos SELinux](#-reverse-proxy-y-booleanos-selinux)
7. [Puertos no estándar (ej. 3000, 5000, 8000)](#-puertos-no-estándar-ej-3000-5000-8000)
8. [Firewalld](#-firewalld)
9. [Direcciones IP: 0.0.0.0 vs 127.0.0.1 vs privada vs pública](#-direcciones-ip-0000-vs-127001-vs-privada-vs-pública)
10. [Servicio systemd para backend Python](#-servicio-systemd-para-backend-python)
11. [Checklist completo de puesta en marcha](#-checklist-completo-de-puesta-en-marcha)
12. [Árbol de decisión: ¿por qué no funciona?](#-árbol-de-decisión-por-qué-no-funciona)
13. [Comandos de referencia rápida](#-comandos-de-referencia-rápida)
14. [Notas propias (tabla de puertos)](#-notas-propias-tabla-de-puertos)
15. [Referencias](#-referencias)

---

## 🗺️ Arquitectura típica

```
Cliente (Internet)
    │
    ▼
┌─────────────────────────────────────────────┐
│  SERVIDOR                                   │
│                                             │
│  Nginx escucha:  0.0.0.0:443  ◄── recibe    │
│       │                                     │
│       │ proxy_pass a 127.0.0.1:3000         │
│       ▼                                     │
│  Backend (gunicorn/uvicorn):                │
│        127.0.0.1:3000  ◄── solo local       │
└─────────────────────────────────────────────┘
```

**Regla de oro:** el backend **nunca** debe escuchar en `0.0.0.0` ni estar
expuesto a internet. Solo Nginx debe ser alcanzable desde fuera.

---

## 🧠 Conceptos clave

| Término | Qué es |
|---|---|
| **SELinux** | Control de acceso obligatorio (MAC). Etiqueta procesos y archivos con "contextos" y decide quién puede tocar qué. |
| **Contexto SELinux** | Etiqueta tipo `usuario:rol:tipo:nivel`. Lo relevante es el **tipo** (ej. `httpd_sys_content_t`). |
| **Dominio SELinux** | Contexto aplicado a un **proceso** en ejecución (ej. `httpd_t` para Nginx). |
| **Booleano SELinux** | Interruptor on/off que activa permisos predefinidos (`httpd_can_network_relay`, etc.). |
| **`semanage fcontext`** | Define **reglas permanentes** de contexto para rutas de archivos. |
| **`restorecon`** | Aplica las reglas de `semanage` a los archivos reales. |
| **`chcon`** | Cambia el contexto manualmente pero **NO es persistente** ante re-etiquetados. Evitarlo. |
| **Firewalld** | Cortafuegos del sistema (control de red). |
| **systemd** | Gestor de servicios. Los demonios se definen en `/etc/systemd/system/*.service`. |

---

## 🔍 Diagnóstico rápido (orden recomendado)

Ejecutar **en este orden** cuando algo falla:

### 1. ¿SELinux está bloqueando algo?

```bash
getenforce                              # Enforcing | Permissive | Disabled
sestatus                                # Detalle completo
sudo ausearch -m avc -ts recent         # Denegaciones recientes
sudo journalctl -t audit -g avc --since "10 min ago"
```

Si hay denegaciones, la salida muestra `tcontext=` (contexto del recurso denegado)
y `scontext=` (contexto del proceso). Comparar con el esperado.

### 2. ¿El backend está vivo y escuchando donde crees?

```bash
sudo systemctl status <servicio>
sudo journalctl -u <servicio> -n 50 --no-pager
sudo ss -tlnp | grep :<puerto>
# Esperado: LISTEN 0 ... 127.0.0.1:3000 ...
# Si aparece 0.0.0.0:3000 → mal, backend expuesto
```

### 3. ¿El backend responde desde la propia máquina?

```bash
curl -v http://127.0.0.1:3000
# Connection refused → backend caído o en otro puerto
# HTTP 200 → backend OK, el problema está en Nginx o firewalld
```

### 4. ¿Nginx llega al backend?

```bash
sudo tail -f /var/log/nginx/error.log
# "connect() failed (111: Connection refused)"  → backend caído
# "connect() failed (13: Permission denied)"    → SELinux bloquea
# "Permission denied" sin contexto SELinux      → DAC (permisos Linux)
```

### 5. ¿Contexto de carpetas y procesos?

```bash
ls -Zd /ruta/web                        # Contexto del directorio
ls -Z  /ruta/web/*                      # Contexto de archivos
ps -eZ | grep nginx                     # Esperado: ... httpd_t ...
ps -eZ | grep -E "gunicorn|uvicorn|python"
```

### 6. ¿Firewalld permite el tráfico?

```bash
sudo firewall-cmd --list-all
# Debe mostrar http y https en "services"
# O los puertos custom en "ports"
```

---

## 🚫 Errores comunes y su causa

| Error | Causa probable | Solución |
|---|---|---|
| **403 Forbidden** | SELinux bloquea lectura o permisos DAC insuficientes | Ver [Carpetas web](#-carpetas-web-fuera-de-las-de-defecto) |
| **404 Not Found** | Ruta mal escrita, archivo no existe | Revisar `root`/`alias` en Nginx |
| **502 Bad Gateway** | Nginx no puede conectar con backend (SELinux, backend caído, puerto mal) | Ver [Reverse proxy](#-reverse-proxy-y-booleanos-selinux) |
| **504 Gateway Timeout** | Backend responde muy lento | Revisar app, `proxy_read_timeout` |
| **Connection refused (curl)** | Backend no escucha en ese puerto | `ss -tlnp`, `systemctl status` |
| **Connection timed out (externo)** | Firewalld bloquea o IP/puerto mal | `firewall-cmd --list-all` |

### Diferencia entre 403, 404 y 502

- **404** → El recurso **no existe** en esa ruta.
- **403** → El recurso **existe**, pero el servidor **rechaza** servirlo
  (permisos DAC, SELinux, `deny all`, etc.).
- **502** → Nginx **sí** recibió la petición, pero **no pudo hablar** con el backend.

---

## 📂 Carpetas web fuera de las de defecto

Cuando el contenido web **no** está en `/var/www/html` (ej. `/data/web`,
`/opt/www`, `/home/user/web`), SELinux asigna por defecto un contexto que Nginx
**no puede leer** → error **403**.

### Contextos que Nginx espera

| Contexto | Uso |
|---|---|
| **`httpd_sys_content_t`** | Contenido que Nginx solo **lee** (HTML, CSS, JS, imágenes) |
| **`httpd_sys_rw_content_t`** | Contenido donde Nginx **escribe** (uploads, caché, logs, SQLite) |
| **`httpd_sys_script_exec_t`** | Scripts ejecutables (CGI) |

### Solución permanente (recomendada)

```bash
# 1. Definir la regla de contexto PERMANENTE
sudo semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"

# 2. Aplicarla a los archivos actuales
sudo restorecon -Rv /data/web

# 3. Verificar
ls -Zd /data/web
# Esperado: ... httpd_sys_content_t ...
```

### ⚠️ Si la app necesita escribir (logs, uploads)

```bash
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/data/web/app(/.*)?"
sudo restorecon -Rv /data/web/app
```

### ❌ NO usar

```bash
sudo chcon -R -t httpd_sys_content_t /data/web   # TEMPORAL, se pierde
sudo chmod -R 777 /data/web                      # NUNCA
setenforce 0                                     # Solo debug puntual
```

### Si el directorio está en el home de un usuario

```bash
sudo setsebool -P httpd_enable_homedirs 1
```

### Diagnóstico del contexto

```bash
sudo semanage fcontext -l | grep "/data/web"     # Reglas definidas
ls -Zd /data/web                                 # Contexto real
sudo ausearch -m avc -ts recent | grep data      # Bloqueos
```

---

## 🔄 Reverse proxy y booleanos SELinux

Nginx se ejecuta en el dominio **`httpd_t`**. Por defecto, SELinux **bloquea**
que `httpd_t` realice conexiones de red salientes → **502 Bad Gateway**.

### Booleanos implicados

| Booleano | Qué permite | Cuándo usarlo |
|---|---|---|
| **`httpd_can_network_relay`** | Proxy hacia puertos etiquetados como `http_port_t` | Recomendado (más seguro) |
| **`httpd_can_network_connect`** | Conexiones a **cualquier** puerto (comodín amplio) | Solo si no puedes etiquetar el puerto |
| `httpd_can_network_connect_db` | Conexión a BD (MySQL, PostgreSQL) | Si Nginx habla directo a una BD |
| `httpd_can_network_connect_proxy` | Proxy saliente genérico | Casos específicos |

### Aplicar

```bash
# Opción recomendada (más segura)
sudo setsebool -P httpd_can_network_relay 1

# Opción amplia (menos segura, pero funciona siempre)
sudo setsebool -P httpd_can_network_connect 1
```

El flag **`-P`** hace el cambio **persistente** tras reinicios.

### Verificar booleanos activos

```bash
getsebool -a | grep httpd
semanage boolean -l | grep httpd
```

### ⚠️ Diferencia conceptual

- **`httpd_can_network_relay`** → "Nginx puede reenviar tráfico a **puertos web**"
  (proxy). Semánticamente correcto para un reverse proxy.
- **`httpd_can_network_connect`** → "Nginx puede conectar a **cualquier** puerto".
  Más amplio, más cómodo, **menos seguro**.

En homelab puedes usar el amplio sin drama. En producción, prefiere el específico
+ etiquetar el puerto.

---

## 🔌 Puertos no estándar (ej. 3000, 5000, 8000)

SELinux solo permite a `httpd_t` conectarse a puertos etiquetados como
**`http_port_t`**. Si el backend usa un puerto no estándar, hay que etiquetarlo.

### Ver puertos actualmente permitidos

```bash
semanage port -l | grep http_port_t
# Verás: 80, 81, 443, 488, 8008, 8009, 8443, 9000
```

### Etiquetar un puerto nuevo (ej. 3000)

```bash
sudo semanage port -a -t http_port_t -p tcp 3000     # añadir
sudo semanage port -m -t http_port_t -p tcp 3000     # modificar
sudo semanage port -d -t http_port_t -p tcp 3000     # eliminar
semanage port -l | grep 3000                         # verificar
```

### Después de etiquetar

```bash
sudo setsebool -P httpd_can_network_relay 1
```

Con esto, **no** necesitas `httpd_can_network_connect`. Nginx podrá hacer
`proxy_pass http://127.0.0.1:3000` sin tocar el permiso amplio.

---

## 🔥 Firewalld

### Comandos esenciales

```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --get-active-zones

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### Regla importante en reverse proxy

- **Nginx** → abre **80/443** (para tráfico externo).
- **Backend** → **NO se abre al exterior**. Solo Nginx (por loopback o IP privada)
  necesita alcanzarlo. Si backend y Nginx están en la misma máquina, el puerto del
  backend **ni siquiera aparece en Firewalld**.

### ¿Y si el backend está en otra máquina de la LAN?

Aquí sí abres el puerto en el **firewalld del backend**, pero restringido a la IP
de Nginx:

```bash
# En el servidor del backend
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="192.168.1.10/32"
  port protocol="tcp" port="3000" accept'
sudo firewall-cmd --reload
```

---

## 🌐 Direcciones IP: 0.0.0.0 vs 127.0.0.1 vs privada vs pública

| Dirección | Significado | Uso típico |
|---|---|---|
| **`127.0.0.1`** | Loopback. Solo accesible desde la propia máquina. | Backend — solo Nginx debe llegar |
| **`0.0.0.0`** | Wildcard de **escucha**: "escucha en todas las interfaces" | Nginx (recibe de internet) |
| **IP privada** (ej. `192.168.1.10`, `10.0.0.5`) | Dirección dentro de la LAN / VPC | Comunicación entre máquinas de la misma red |
| **IP pública** (ej. `203.0.113.45`) | Dirección visible desde internet | Nginx en el borde (o NAT del router) |

### Reglas de oro

1. **Backend → `127.0.0.1`** (mismo host) o **IP privada** (otro host de la LAN).
2. **Nginx → `0.0.0.0`** o IP pública.
3. **Nunca backend en `0.0.0.0`** si no quieres exponerlo. Si escucha ahí, cualquiera
   que pueda alcanzar el puerto habla directo con tu app, **saltando Nginx**
   (sin rate limiting, sin TLS, sin WAF, sin logs, sin auth).

### Cómo verificar dónde escucha un proceso

```bash
sudo ss -tlnp | grep :<puerto>
```

- `127.0.0.1:3000` → solo loopback ✅
- `0.0.0.0:3000`   → todas las interfaces ⚠️
- `*:3000`         → todas las interfaces (IPv4+IPv6) ⚠️

### Flujo típico

```
Cliente (internet)
    │ conecta a midominio.com → resuelve a IP pública
    ▼
Nginx escucha 0.0.0.0:443
    │ proxy_pass → 127.0.0.1:3000
    ▼
Backend escucha 127.0.0.1:3000
```

El cliente **nunca ve** ni la IP privada ni el puerto 3000.

---

## ⚙️ Servicio systemd para backend Python

### ¿Por qué no usar `python3 -m http.server` en producción?

| Problema | Consecuencia |
|---|---|
| Monohilo | Una petición a la vez → lentitud |
| Sin reinicio automático | Si cae, se queda caído |
| Atado a la terminal | Al cerrar SSH, muere |
| Sin workers | No aprovecha múltiples núcleos |
| Sin gestión de señales | No se apaga limpiamente |

Sirve para **probar** que Nginx llega, pero no como solución final.

### Stack recomendado

| Framework | Servidor | Comando |
|---|---|---|
| Flask / Django (sync) | **gunicorn** | `gunicorn --bind 127.0.0.1:3000 miapp.wsgi:application` |
| FastAPI / Django (async) | **uvicorn** | `uvicorn miapp.asgi:application --host 127.0.0.1 --port 3000` |
| Flask (async) | **hypercorn** | similar a uvicorn |

### Crear usuario de sistema

```bash
sudo useradd --system --no-create-home --shell /sbin/nologin miapp
```

### Archivo `/etc/systemd/system/miapp.service`

```ini
[Unit]
Description=Mi aplicación Python (gunicorn)
After=network.target

[Service]
Type=notify
User=miapp
Group=miapp
WorkingDirectory=/data/web/miapp
Environment="PATH=/data/web/miapp/venv/bin"
ExecStart=/data/web/miapp/venv/bin/gunicorn \
    --workers 3 \
    --bind 127.0.0.1:3000 \
    --access-logfile - \
    --error-logfile - \
    miapp.wsgi:application

Restart=on-failure
RestartSec=5s

# Endurecimiento básico
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

### Activar y arrancar

```bash
sudo systemctl daemon-reload            # tras crear/modificar el .service
sudo systemctl enable --now miapp       # habilitar al boot + arrancar ahora

systemctl is-enabled miapp              # → enabled
systemctl is-active miapp               # → active
systemctl status miapp
sudo journalctl -u miapp -f             # logs en vivo
```

### Diferencias de comandos systemd

```bash
sudo systemctl enable miapp        # arranca al boot
sudo systemctl start miapp         # arranca ahora
sudo systemctl enable --now miapp  # ambos

sudo systemctl disable miapp       # no arranca al boot
sudo systemctl stop miapp          # para ahora
sudo systemctl restart miapp       # para y arranca
sudo systemctl reload miapp        # recarga config (si soportado)
```

### ⚠️ SELinux también afecta a systemd

El binario y su directorio de trabajo deben tener **contexto correcto** para que
systemd lo ejecute en el dominio adecuado:

```bash
# Contenido estático
sudo semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"

# Directorio donde la app escribe (logs, uploads, SQLite, etc.)
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/data/web/miapp(/.*)?"

sudo restorecon -Rv /data/web

# Verificar dominio del proceso
ps -eZ | grep gunicorn
# Esperado: system_u:system_r:httpd_t:s0 ...
```

Si aparece `unconfined_service_t` u otro dominio raro, revisar:
- Contexto del binario (`ls -Z /data/web/miapp/venv/bin/gunicorn`)
- Contexto del directorio de trabajo
- Que el `.service` no tenga `SELinuxContext=` forzando algo raro

---

## ✅ Checklist completo de puesta en marcha

### Fase 1 — Preparación

- [ ] Elegir puerto libre para backend (`ss -tlnp | grep :3000`)
- [ ] Crear directorio de la app (`/data/web/miapp`)
- [ ] Crear usuario de sistema (`useradd --system miapp`)
- [ ] Copiar código y crear virtualenv
- [ ] Instalar gunicorn/uvicorn en el venv

### Fase 2 — SELinux (contextos de archivos)

- [ ] `semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"
- [ ] `semanage fcontext -a -t httpd_sys_rw_content_t "/data/web/miapp(/.*)?"`
- [ ] `restorecon -Rv /data/web`
- [ ] Verificar: `ls -Zd /data/web` muestra `httpd_sys_content_t`

### Fase 3 — SELinux (puertos y booleanos)

- [ ] `semanage port -a -t http_port_t -p tcp 3000`
- [ ] `setsebool -P httpd_can_network_relay 1`
- [ ] Verificar: `semanage port -l | grep 3000` → `http_port_t`
- [ ] Verificar: `getsebool httpd_can_network_relay` → `on`

### Fase 4 — Servicio systemd

- [ ] Crear `/etc/systemd/system/miapp.service`
- [ ] `systemctl daemon-reload`
- [ ] `systemctl enable --now miapp`
- [ ] Verificar: `systemctl status miapp` → `active (running)`
- [ ] Verificar: `ss -tlnp | grep 3000` → `127.0.0.1:3000`
- [ ] Probar: `curl -v http://127.0.0.1:3000` → `HTTP 200`

### Fase 5 — Nginx

- [ ] Configurar `server` block con `proxy_pass http://127.0.0.1:3000`
- [ ] `nginx -t` → sintaxis OK
- [ ] `systemctl reload nginx`
- [ ] Probar: `curl -k https://midominio.local` desde la LAN

### Fase 6 — Firewalld

- [ ] `firewall-cmd --permanent --add-service=http`
- [ ] `firewall-cmd --permanent --add-service=https`
- [ ] `firewall-cmd --reload`
- [ ] **NO** abrir el puerto 3000 al exterior
- [ ] Verificar: `firewall-cmd --list-all` → `http https`

### Fase 7 — Verificación final

- [ ] `curl` local al backend → OK
- [ ] `curl` al Nginx local → OK
- [ ] `curl` desde otra máquina de la LAN → OK
- [ ] `curl` desde internet (si aplica) → OK
- [ ] Logs de Nginx sin errores: `tail -f /var/log/nginx/error.log`
- [ ] `ausearch -m avc -ts recent` → sin denegaciones nuevas

---

## 🩺 Árbol de decisión: ¿por qué no funciona?

```
¿Falla desde fuera?
├─ Sí → ¿Funciona desde dentro con curl a 127.0.0.1?
│       ├─ No → problema del backend (systemd, puerto, app)
│       └─ Sí → problema de Nginx o firewalld
│               ├─ ¿nginx -t OK? ¿systemctl status nginx? → activo
│               ├─ ¿firewall-cmd --list-all tiene http/https?
│               └─ ¿firewall-cmd --add-port necesario?
│
└─ No → el problema es puntual:
        ├─ 403 → contexto SELinux o permisos DAC
        ├─ 404 → ruta en Nginx
        ├─ 502 → SELinux (puerto no etiquetado / booleano off) o backend caído
        └─ 504 → backend lento
```

---

## 🎛️ Comandos de referencia rápida

### SELinux — estado y diagnóstico

```bash
getenforce                              # Enforcing | Permissive | Disabled
sestatus                                # Detalle completo
ausearch -m avc -ts recent              # Denegaciones recientes
ausearch -m avc -ts today | audit2why   # Explicación humana de bloqueos
journalctl -t audit -g avc --since "10 min ago"
setenforce 0                            # PERMISIVO temporal (⚠️ solo debug)
setenforce 1                            # Volver a Enforcing
```

### SELinux — contextos de archivos

```bash
semanage fcontext -a -t httpd_sys_content_t "/ruta(/.*)?"
semanage fcontext -l | grep /ruta       # Ver reglas definidas
restorecon -Rv /ruta                    # Aplicar reglas
ls -Zd /ruta                            # Ver contexto actual
ls -Z  /ruta/*                          # Ver contexto de archivos
```

### SELinux — puertos

```bash
semanage port -l | grep http_port_t     # Puertos permitidos como web
semanage port -a -t http_port_t -p tcp 3000
semanage port -m -t http_port_t -p tcp 3000
semanage port -d -t http_port_t -p tcp 3000
```

### SELinux — booleanos

```bash
getsebool -a | grep httpd               # Ver booleanos httpd
semanage boolean -l | grep httpd        # Detalle con descripción
setsebool -P httpd_can_network_relay 1
```

### Procesos y puertos

```bash
ss -tlnp                                # Puertos TCP escuchando
ss -tlnp | grep :3000                   # Verificar puerto específico
ps -eZ | grep nginx                     # Contexto SELinux de Nginx
ps -eZ | grep gunicorn                  # Contexto de gunicorn
```

### systemd

```bash
systemctl daemon-reload                 # Tras editar .service
systemctl enable --now miapp            # Habilitar + arrancar
systemctl status miapp                  # Estado
systemctl restart miapp                 # Reiniciar
journalctl -u miapp -f                  # Logs en vivo
journalctl -u miapp -n 100              # Últimas 100 líneas
```

### Firewalld

```bash
firewall-cmd --state                    # ¿Está corriendo?
firewall-cmd --list-all                 # Reglas de la zona activa
firewall-cmd --get-active-zones
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload                   # Aplicar cambios permanentes
```

### Nginx

```bash
nginx -t                                # Validar config
systemctl reload nginx                  # Recargar sin cortar conexiones
tail -f /var/log/nginx/error.log        # Errores en vivo
tail -f /var/log/nginx/access.log       # Accesos en vivo
```

### curl — pruebas

```bash
curl -v http://127.0.0.1:3000           # Backend local
curl -v http://127.0.0.1                # Nginx local (HTTP)
curl -vk https://127.0.0.1              # Nginx local (HTTPS autofirmado)
curl -v http://192.168.1.10:3000        # Backend desde otra máquina de la LAN
curl -v https://midominio.com           # Desde internet
```

---

## 📝 Notas propias (tabla de puertos)

_(Editar con los puertos y servicios del homelab personal)_

| Puerto | Servicio | Host/bind | Dominio |
|---|---|---|---|
| 80 | Nginx | `0.0.0.0` | — |
| 443 | Nginx | `0.0.0.0` | — |
| 3000 | App recetas | `127.0.0.1` | `recetas.local` |
| 3001 | App notas | `127.0.0.1` | `notas.local` |
| 5432 | PostgreSQL | `127.0.0.1` | — |
| ... | ... | ... | ... |

---

## 📚 Referencias

- [RHEL SELinux — Managing confined services](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/using_selinux/)
- [Nginx — Reverse proxy guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Firewalld — Documentation](https://firewalld.org/documentation/)
- [systemd.service man page](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [Gunicorn — Deploying](https://docs.gunicorn.org/en/stable/deploy.html)
- [Uvicorn — Deployment](https://www.uvicorn.org/deployment/)

---

**Última actualización:** _(fecha)_
**Autor:** _(tu nombre / usuario)_