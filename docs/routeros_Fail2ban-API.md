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
```
### firewall-externo.conf
```
[Definition]
actionstart =
actionstop =
actioncheck =
actionban = /ruta/a/bloquear_mikrotik.py <ip>
actionunban =
```

## 🧠 Nota de diseño: no se define actionunban, ya que Mikrotik gestiona el desbloqueo automáticamente mediante listas dinámicas con timeout. Esto permite una sincronización natural con el bantime de Fail2Ban, evitando redundancias y manteniendo la arquitectura desacoplada.

## 🐍 Script bloquear_mikrotik.py
```
#!/home/eferro/venvs/mikrotik-api/bin/python3
from routeros_api import RouterOsApiPool
import sys

if len(sys.argv) != 2:
    print("Uso: bloquear_mikrotik.py <IP>")
    sys.exit(1)

# IP a bloquear (recibida desde Fail2Ban)
ip_bloquear = sys.argv[1]

# Conexión al Mikrotik
api_pool = RouterOsApiPool(
    host='10.10.10.254',
    username='mngapi',
    password='XxCv15IzslLjnwm73)x',
    plaintext_login=True
)
try:
    api = api_pool.get_api()

    # Agregar IP a la lista negra
    firewall = api.get_resource('/ip/firewall/address-list')
    firewall.add(
        address=ip_bloquear,
        list='spammer',
        timeout='00:30:00',
        comment='Desde Fail2Ban',
        disabled='no'
    )
    api_pool.disconnect()
except Exception as e:
    print(f"Error al conectar con Mikrotik: {e}")
    sys.exit(1)

if __name__ == "__main__":
    print(f"IP {ip_bloquear} bloqueada en Mikrotik.")
```

## ✅ Validación
En Debian
sudo fail2ban-client status auth_fail


sudo fail2ban-client status auth_fail | grep 'Banned IP list'


En Mikrotik
/ip firewall address-list print where list="Spammer"



## 🧠 Filosofía de diseño distribuido
- Fail2Ban: análisis local, detección y control de tiempo.
- Mikrotik: ejecución remota, bloqueo y desbloqueo automático.
- Sincronización por diseño: bantime = timeout, sin necesidad de scripts de unban.
- Desacoplamiento limpio: cada sistema cumple su rol sin interferencias.

## 📦 Requisitos
- Python 3
- routeros_api (pip install routeros_api)
- Acceso SSH a Debian y API habilitada en Mikrotik
- Logs remotos configurados en Mikrotik (/system logging action remote)

## 🧪 Casos de prueba
- IP que falla 3 veces en 10 minutos → bloqueada 30 minutos
- IP ya bloqueada → script ignora duplicados
- IP desbloqueada automáticamente por Mikrotik → Fail2Ban no interviene

## 📝 Notas adicionales

- Este sistema puede escalarse a múltiples Mikrotik con mínima modificación.
- Se recomienda monitoreo con bpytop, Conky o scripts de log rotativo.
- Ideal para entornos con múltiples puntos de entrada y logs centralizados.

### 🤝 Autor
Ezequiel — Especialista en sistemas, redes y ciberseguridad. Documentación reproducible, scripting quirúrgico y ciencia personal aplicada.




