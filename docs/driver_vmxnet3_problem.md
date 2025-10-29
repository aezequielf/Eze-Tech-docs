# 🧠 Diagnóstico de congelamiento aleatorio en VM Linux sobre ESXi

## 🎯 Contexto

- **Entorno**: Máquina virtual Linux corriendo sobre **VMware ESXi**.
- **Síntoma inicial**: Congelamientos aleatorios del sistema sin *kernel panic* ni registros en `syslog`, `dmesg` ni `kern.log`.
- **Evento previo**: Un único *kernel panic* registrado: Kernel panic - not syncing: Fatal exception in interrupt
  
## 🔍 Hipótesis

> El freeze podría estar relacionado con el **driver de red virtual** `vmxnet3`, conocido por causar fallos intermitentes en ciertos kernels o entornos VMware.

## 🧪 Validación experimental

1. **Revisión exhaustiva de logs **: No se encontraron trazas útiles post-freeze.
2. **Activación de `kdump`**: Se configuró para capturar volcados en caso de futuros *panics*. No se sucedió posteriormente ninguno, solo freez sin logs.
3. **Cambio de driver de red**:
 - Se reemplazó `vmxnet3` por `e1000` en la configuración de la VM.
 - Desde entonces, el sistema se mantuvo **estable y sin congelamientos**.

## ✅ Conclusión

El cambio de driver validó la hipótesis: `vmxnet3` era el probable causante del freeze. Aunque no se generó un nuevo *panic*, el comportamiento se estabilizó con `e1000`.

## 📦 Recomendaciones

- Mantener `kdump` activo para capturar futuros eventos.
- Documentar el cambio en la configuración de red de la VM.
- Considerar volver a `vmxnet3` solo si se actualiza el kernel o VMware Tools.
- Monitorear el sistema con herramientas como `sar`, `iotop`, `vmstat`.

---

> 🧩 Este caso fue documentado como parte de una bitácora de diagnóstico experimental, con enfoque en reproducibilidad, trazabilidad y transferencia de conocimiento técnico.
