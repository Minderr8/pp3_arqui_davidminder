# Día 1 — Setup e instalación (resumen)

## ✅ Checklist de hoy
- [x] Ollama instalado
- [x] gemma3:4b descargado y probado
- [x] GPU detectada automáticamente (ROCm sin configurar nada)
- [x] Herramientas de monitoreo instaladas y probadas (sensors, turbostat)
- [ ] rocm-smi (opcional, no instalado aún)
- [ ] Experimentos 1, 2, 3 (mañana)

---

## 📄 PARA EL INFORME (informe.md)

Esto es texto/datos que puedes copiar casi directo a las secciones correspondientes.

### Sección `## Hardware`
```
CPU: AMD Ryzen 7 5700X, 8 núcleos / 16 hilos, hasta 4.67 GHz (boost)
RAM: ~31 GB total
GPU: AMD Radeon RX 7700S (Navi 33, arquitectura RDNA3)
Placa madre: ASRock B550 Phantom Gaming 4
Almacenamiento: NVMe (detectado en sensors)
Sistema operativo: Ubuntu (kernel 6.17.0-29-generic)
Tipo de equipo: Torre de escritorio (no laptop, pese a que la RX 7700S 
                es un chip normalmente orientado a laptops)
```

### Sección `## Modelo`
```
Modelo: gemma3:4b
Cuantización: Q4_K_M (default de Ollama)
Tamaño en disco: 2.9 GB (confirmado con `ollama list` / `ollama ps`)
Justificación: se usó el modelo recomendado sin modificaciones, 
               ya que el hardware lo soporta sin problema en GPU.
```

### Sección `## Metodología` — herramientas confirmadas
```
- Detección de GPU: `ollama ps` → confirma split CPU/GPU (100% GPU sin configurar HSA_OVERRIDE_GFX_VERSION)
- Temperatura y watts GPU: `sensors` (driver amdgpu) → edge, junction, mem, PPT (watts reales)
- Temperatura CPU: `sensors` (driver k10temp) → Tctl, Tccd1
- Watts y frecuencia CPU: `sudo turbostat --interval 2` → columna PkgWatt (consumo total del package)
- Forzar CPU-only (pendiente de usar mañana): CUDA_VISIBLE_DEVICES="" ollama run ...
```

### Dato clave — Baseline en reposo (sin inferencia corriendo)
Útil como punto de comparación en la sección de Resultados/Análisis:
```
GPU en reposo:
  PPT: 6.00 W (cap 145 W)
  edge/junction: 37°C | mem: 44°C

CPU en reposo (turbostat):
  PkgWatt: ~46-50 W
  Tctl: 59.2°C

Este baseline permite calcular el delta de consumo real atribuible 
a la inferencia (Watts_inferencia - Watts_reposo).
```

### Observación para `## Análisis` (anótala para no olvidarla)
La GPU funcionó al 100% sin necesitar `HSA_OVERRIDE_GFX_VERSION` — vale la pena 
mencionar esto como hallazgo, ya que RDNA3 "gaming" normalmente tiene soporte 
oficial limitado en ROCm y se esperaría más fricción.

---

## 🧠 PARA TI (aprendizaje, no va al informe)

### Ollama
- Motor que descarga, cuantiza (si hace falta) y sirve modelos LLM localmente
- Detecta automáticamente GPU (NVIDIA/AMD/Intel) sin configuración extra en tu caso
- `ollama pull <modelo>` descarga, `ollama run <modelo> "prompt"` ejecuta, 
  `ollama ps` muestra qué está cargado y dónde (CPU/GPU), `ollama list` lista lo descargado

### Cuantización
- Los pesos de un modelo normalmente están en float16/32
- Cuantizar = comprimir esos pesos a menor precisión (ej. 4 bits) para 
  ahorrar RAM/VRAM y ganar velocidad, a cambio de algo de precisión
- Q4_K_M = 4 bits por peso, variante K_M de compresión (buen balance calidad/tamaño)

### ROCm (equivalente AMD de CUDA)
- Stack de AMD para cómputo paralelo en GPU (no solo gráficos)
- Soporte oficial típicamente mejor en tarjetas datacenter que en gaming/laptop
- En tu caso funcionó "out of the box", lo cual no era garantizado

### sensors / lm-sensors
- Lee sensores de hardware (temperatura, voltaje, watts) directo del kernel
- `sensors-detect` detecta qué chips de sensores tiene tu placa y carga los drivers necesarios
- Tras detectarlos, el comando `sensors` los muestra en texto plano
- `amdgpu-pci-XXXX` = tu GPU, `k10temp-pci-XXXX` = tu CPU (Ryzen)
- PPT (Package Power Tracking) = consumo real en watts, la métrica de oro para tu proyecto

### turbostat
- Herramienta de Linux (viene en linux-tools) que lee contadores internos del CPU
- Te da frecuencia real por núcleo (Bzy_MHz), % de uso (Busy%), y consumo (CorWatt/PkgWatt)
- PkgWatt = consumo total del "package" (todo el chip CPU), la métrica clave para tu proyecto
- Se corre con `--interval N` (cada cuántos segundos actualiza) y opcionalmente `--out archivo.log`

### rocm-smi (pendiente, opcional)
- Herramienta específica de AMD, similar a nvidia-smi
- Daría métricas más finas de GPU: % de uso real, VRAM usada, clocks
- No es obligatorio porque `sensors` ya cubre temperatura y watts de la GPU
- Se instala con `sudo apt install rocm-smi`

---

## 🎯 Plan para mañana (Experimentos)
1. Definir estructura de carpetas del entregable (`apellido_nombre/`)
2. Para cada experimento (1, 2, 3):
   - Correr en GPU (default) → 3 veces, capturar tokens/s, tiempo, screenshot
   - Forzar CPU con `CUDA_VISIBLE_DEVICES=""` → 3 veces, capturar igual
   - En paralelo (otra terminal): `sensors` y `turbostat --out archivo.log` corriendo durante la generación
3. Armar `metrics.json` con los promedios de las 3 corridas por experimento/dispositivo
4. Empezar a redactar `informe.md` con los datos ya casi listos de este resumen
