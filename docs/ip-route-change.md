# 🧭 Modificar rutas en Linux con `ip route change`: más allá del delete/add

## 📌 Introducción

En la mayoría de los tutoriales sobre administración de rutas en Linux, el enfoque habitual para modificar una ruta existente es eliminarla (`ip route delete`) y luego agregarla nuevamente (`ip route add`) con los nuevos parámetros. Aunque funcional, este método tiene limitaciones:

- Puede generar breves interrupciones de conectividad.
- No preserva atributos como métricas, protocolos o scopes.
- No es idempotente ni seguro en entornos automatizados.

Existe una alternativa más precisa: `ip route change`. Sin embargo, su uso requiere entender cómo el kernel valida las rutas existentes. Esta documentación explora su funcionamiento y casos de uso.

---

## 🧠 ¿Cómo funciona `ip route change`?

El comando `ip route change` **no modifica parcialmente** una ruta. En cambio, busca una coincidencia **exacta** en la tabla de rutas. Si todos los atributos coinciden, reemplaza los valores que se le indiquen (por ejemplo, el gateway). Si no hay coincidencia perfecta, el kernel devuelve:

```bash
RTNETLINK answers: No such file or directory
```
---

## 🔍 Atributos que deben coincidir

Para que el cambio sea aceptado, deben coincidir todos los atributos de la ruta existente:

- Destino ()
- Interfaz ()
- Gateway ()
- Métrica ()
- Protocolo (, , etc.)
- Scope (, )
- Tabla (, , )

---

## 🧪 Ejemplo práctico

Supongamos que tenés esta ruta:
```bash
ip route show 192.168.1.0/24

192.168.1.0/24 via 192.168.1.1 dev eth0 proto static metric 100
```
Querés cambiar el gateway a 192.168.1.254. El comando correcto sería:
```bash
sudo ip route change 192.168.1.0/24 via 192.168.1.254 dev eth0 proto static metric 100
```
---

## NOTA, si no colocaras dev eth0 el kernel intenta deducir este parámetro de acuerdo a su tabla de rutas directamente conectadas a la interfaz.

---

## 🧷 Ventajas de
 
- No interrumpe la conectividad.
- Preserva atributos avanzados como métricas, protocolos y scopes.
- Ideal para scripts de monitoreo y corrección automática.
- Compatible con entornos críticos y rutas dinámicas.
- Evita efectos colaterales en tablas de rutas sensibles.

---

## 📚 Conclusión
 `ip route change` es una herramienta poderosa y subdocumentada. Su uso correcto requiere comprender cómo el kernel valida las rutas, pero ofrece una alternativa más segura y elegante que el clásico delete/add. En entornos donde la precisión importa —como redes críticas, automatización o debugging forense— este enfoque marca la diferencia.

## ✍️ Autor
Documentado por Ezequiel, especialista en sistemas y redes. Este artículo forma parte de su bitácora técnica sobre ciencia personal, scripting y troubleshooting reproducible.




