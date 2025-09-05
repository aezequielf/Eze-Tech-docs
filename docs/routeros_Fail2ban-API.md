# 🛡️ Integración Fail2Ban + Mikrotik vía API 

Sistema distribuido de defensa automatizada que detecta intentos de autenticación fallida en logs del sistema y bloquea las IP ofensivas directamente en el firewall de RouterOS, usando listas dinámicas con timeout sincronizado. Esta mini tutorial supone el manejo de ciertos conocimientos básicos de Linux, Mikrotik, redes y programación python. No entraremos en detalles de como se implementa el firewall en RouterOs para que use la lista dinámica ya que eso conlleva un proceso menor y sencillo. Tampoco observaremos la creación de un usuario que tenga permisos de aceptar y ejecutar las peticiones de la api en RouterOS.

---

## 🎯 Objetivo

- Detectar intentos de acceso fallido en logs.
- Bloquear automáticamente las IP ofensivas en Mikrotik.
- Mantener sincronización entre `bantime` de Fail2Ban y `timeout` de la lista dinámica.
- Evitar redundancias en el proceso de desbloqueo.

---
## ⚠ Instalar la librería routeros_api de python

Necesitamos instalar una librería especial que nos resuelve ya la interacción con la API de RouterOS. Esta librería cuando la instalamos con pip, nos genera un error en la que para resoverlo lo más sencillo es crear un entorno virtual. En Linux no tenemos ni pip ni venv instalados por defecto así que ...
```
sudo atp install pip python3.11-venv
```
#### Nota: estoy usando justo python 3.11

Una vez hecho esto:
```
python3 -m venv ~/venvs/mikrotik-api
source venvs/mikrotik-api/bin/activate
pip install routeros_api
```
A esta altura deberíamos tener la librería routero_api instaladas ya dentro de la carpeta del usuario actual en el entorno virtual

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
### auth_fail.conf

```ini
[Definition]
failregex = ^.*\[\s*<HOST>\s*\].*Error: authentication failed
ignoreregex =
```
####  Nota: en esta caso en particular busco en los logs, una linea como esta, donde el ip que quiero capturar está entre los corchetes [] 
```
2025-09-05T11:17:32.937307-03:00 server_auth VPS-uIMVkzSz[212.11.64.212] 1757081850-967898-29275-965059-1 1757081851 1757081852 RECV - - 2 83 535 Error: authentication failed [usuario_que_no_existe]
```
Podemos probar si la expresión regular realmente es correcta y "detecta" la linea que buscamos en el log 

```
 sudo fail2ban-regex /var/log/syslog /etc/fail2ban/filter.d/auth_fail.conf
```
Si funciona bien deberíamos ver unas línas así con el matches distinto de 0, en mi caso matcheó 32243 veces.
``` 
Lines: 78790 lines, 0 ignored, 32243 matched, 46547 missed
[processed in 4.77 sec]
```

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
actionban = /home/usuario_actual/bloquear_mikrotik.py <ip>
actionunban =
```

#### 🧠 Nota de diseño: no se define actionunban, ya que al usar "timeout" en la lista, el item se vuelve dinámico y Routeros gestiona el desbloqueo automáticamente. Esto permite una sincronización natural con el bantime de Fail2Ban, evitando redundancias y manteniendo la arquitectura desacoplada y sincronizada.
#### Como notarás en el script siguiente, en el hashbang, invoco el interprete de python que está dentro del entorno virtual, si no lo hago de esta manera, no podré llamar el módulo de la librería routeros_api.

## 🐍 Script bloquear_mikrotik.py
```
#!/home/usuario_actual/venvs/mikrotik-api/bin/python3
from routeros_api import RouterOsApiPool
import sys

# si el script necesita el ip para la regal de bloqueo 
if len(sys.argv) != 2:
    print("Uso: bloquear_mikrotik.py <IP>")
    sys.exit(1)

# IP a bloquear (recibida desde Fail2Ban)
ip_bloquear = sys.argv[1]

# Conexión al Mikrotik
api_pool = RouterOsApiPool(
    host='ip_del_routeros',
    username='usuario_api_routeros',
    password='clave',
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
Seguramente querrás probar tu script, para verificar que funcione correctamente. Para que tengas en cuenta este script supone que le vas a pasar una ip con formato correcto, no valida esta condición. No te olvides de darle permisos de ejecución 😅

```bash
chmod +x bloquear_mikrotik.py
```
y a probar !!!

```bash
~/bloquear_mikrotik.py 4.4.4.4
```
La respuesta debería ser si todo está bien

```bash
IP 4.4.4.4 bloqueada en Mikrotik.
```
Ahora deberías ver en el routeros un item dinámico nuevo en el address list del firewall que tendrías que usar como source de una regla drop. 

## 🔄 Reinicio del servicio Fail2Ban
Después de configurar el filtro, la acción y el jail, es necesario reiniciar el servicio para aplicar los cambios:
```
bash
sudo systemctl restart fail2ban
```
## ✅ Validación

Verificar que el jail esté activo con:
```bash
sudo fail2ban-client status
```

Y para ver el estado específico del jail:
```bash
sudo fail2ban-client status auth_fail
```


## 🧠 Filosofía de diseño distribuido
- Fail2Ban: análisis local, detección y control de tiempo.
- Mikrotik: ejecución remota, bloqueo y desbloqueo automático.
- Sincronización por diseño: bantime = timeout, sin necesidad de scripts de unban.
- Desacoplamiento limpio: cada sistema cumple su rol sin interferencias.

## 📦 Requisitos
- Python 3
- routeros_api (pip install routeros_api)
- Acceso SSH a Debian y API habilitada en Mikrotik

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




