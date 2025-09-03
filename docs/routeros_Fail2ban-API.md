# 🛡️ Integración Fail2Ban + Mikrotik vía API

Sistema distribuido de defensa automatizada que detecta intentos de autenticación fallida en logs del sistema y bloquea las IP ofensivas directamente en el firewall Mikrotik, usando listas dinámicas con timeout sincronizado.

---

## 🎯 Objetivo

- Detectar intentos de acceso fallido en logs remotos.
- Bloquear automáticamente las IP ofensivas en Mikrotik.
- Mantener sincronización entre `bantime` de Fail2Ban y `timeout` de la lista dinámica.
- Evitar redundancias en el proceso de desbloqueo.

---

## 🧩 Componentes

| Archivo / Script                     | Rol                                                                 |
|-------------------------------------|----------------------------------------------------------------------|
| `filter.d/auth_fail.conf`           | Filtra líneas con `authentication failed` y extrae IP                |
| `action.d/firewall-externo.conf`    | Ejecuta script Python que bloquea IP en Mikrotik                     |
| `jail.local`                        | Define parámetros de detección y ban temporal                        |
| `bloquear_mikrotik.py`              | Script que conecta vía API y agrega IP a lista dinámica `Spammer`   |
| `/var/log/syslog`                   | Fuente de eventos (recibe logs remotos desde Mikrotik)              |

---

## ⚙️ Configuración

### `jail.local`

```ini
[auth_fail]
enabled = true
filter = auth_fail
action = firewall-externo
logpath = /var/log/syslog
maxretry = 3
findtime = 600
bantime = 1800  # 30 minutos
