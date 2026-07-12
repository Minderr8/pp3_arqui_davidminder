# Experimento 1 — Generación de Texto: Resumen y Cierre

## ✅ Estado: COMPLETO
3 corridas GPU + 3 corridas CPU, todas con turbostat corriendo en paralelo.

---

## 📄 PARA EL INFORME (informe.md / metrics.json)

### Resultados — Tabla comparativa

**GPU (gemma3:4b — AMD Radeon RX 7700S, 100% GPU confirmado con `ollama ps`)**

| Run | total duration | eval count (tokens) | eval rate (tok/s) | prompt eval rate (tok/s) |
|---|---|---|---|---|
| 1 | 19.56s | 1218 | 63.82 | 730.22 |
| 2 | 22.35s | 1392 | 63.77 | 1467.43 |
| 3 | 20.30s | 1275 | 64.30 | 699.18 |
| **Promedio** | **20.74s** | **1295** | **63.96** | **965.61** |

**CPU (gemma3-cpu — AMD Ryzen 7 5700X, 100% CPU confirmado, `num_gpu=0`)**

| Run | total duration | eval count (tokens) | eval rate (tok/s) | prompt eval rate (tok/s) |
|---|---|---|---|---|
| 1 | 109.58s | 1241 | 11.43 | 105.61 |
| 2 | 119.70s | 1351 | 11.38 | 105.28 |
| 3 | 108.87s | 1230 | 11.36 | 408.95 |
| **Promedio** | **112.72s** | **1274** | **11.39** | **206.61** |

**Speedup GPU vs CPU:**
- Tokens/s (generación): **~5.6x** más rápido en GPU
- Tiempo total: **~5.4x** más rápido en GPU

### JSON listo para `metrics.json`
```json
"exp1_text": {
  "gpu": {
    "runs": [
      {"tokens_per_sec": 63.82, "total_time_sec": 19.56},
      {"tokens_per_sec": 63.77, "total_time_sec": 22.35},
      {"tokens_per_sec": 64.30, "total_time_sec": 20.30}
    ],
    "average": {"tokens_per_sec": 63.96, "total_time_sec": 20.74}
  },
  "cpu": {
    "runs": [
      {"tokens_per_sec": 11.43, "total_time_sec": 109.58},
      {"tokens_per_sec": 11.38, "total_time_sec": 119.70},
      {"tokens_per_sec": 11.36, "total_time_sec": 108.87}
    ],
    "average": {"tokens_per_sec": 11.39, "total_time_sec": 112.72}
  }
}
```
*(RAM/VRAM y watts/temp promedio pendientes de extraer de los logs de turbostat — ver sección "Pendientes" abajo)*

### Metodología — hallazgo clave para documentar

**Problema:** Forzar CPU-only en Ollama no funcionó con las variables de entorno estándar.

**Intentos fallidos:**
1. `CUDA_VISIBLE_DEVICES=""` → no aplica, es variable de NVIDIA/CUDA, y el hardware es AMD/ROCm
2. `ROCR_VISIBLE_DEVICES=""` → no funcionó
3. `HIP_VISIBLE_DEVICES=""` → tampoco funcionó

**Causa raíz:** Ollama corre como servicio systemd persistente (`ollama serve`, ver `systemctl status ollama`). Las variables de entorno seteadas en la terminal antes de `ollama run` solo aplican al proceso cliente, no al servidor que ya estaba corriendo desde antes y que hace la inferencia real.

**Solución robusta:** Crear un modelo derivado vía Modelfile con el parámetro `num_gpu 0`, que fuerza 0 capas en GPU sin depender de variables de entorno de ningún fabricante:
```
FROM gemma3:4b
PARAMETER num_gpu 0
```
```bash
ollama create gemma3-cpu -f Modelfile-cpu
```
Verificado con `ollama ps` → `100% CPU`.

### Análisis — 3 puntos para la sección más importante del informe

1. **Speedup ~5.6x**: arquitectura SIMT masivamente paralela de la GPU (miles de núcleos simples) vs. pocos núcleos de propósito general en CPU, para la carga de multiplicación de matrices que domina la inferencia.
2. **Consistencia intra-dispositivo**: el eval rate varía <1% entre corridas del mismo dispositivo (63.77–64.30 GPU; 11.36–11.43 CPU) → la medición es confiable y reproducible, no hay ruido significativo afectando el benchmark.
3. **Cuello de botella de un solo hilo (Amdahl)**: en turbostat durante las corridas GPU, se observó un único core de CPU saltando a ~100% mientras el resto permanecía bajo — evidencia del trabajo secuencial no paralelizable (sampling de tokens, tokenización) que ni la GPU puede acelerar. Caso real de la Ley de Amdahl con datos propios.

---

## 🧠 PARA TI (aprendizaje personal)

- **`num_gpu`**: parámetro de Ollama que controla cuántas capas del modelo se cargan en GPU. En 0, fuerza CPU-only sin importar el fabricante.
- **Ollama como servicio systemd**: `ollama serve` corre persistente en background desde que arranca el sistema (o desde la instalación). Por eso las variables de entorno de una sesión de terminal nueva no lo afectan — ya estaba corriendo antes.
- **CUDA vs ROCm vs HIP**: `CUDA_VISIBLE_DEVICES` es específico de NVIDIA. `ROCR_VISIBLE_DEVICES` y `HIP_VISIBLE_DEVICES` son los equivalentes en el stack AMD — pero ninguno sirve si el proceso que lee la variable ya estaba corriendo.
- **prompt eval rate vs eval rate**: procesar el prompt de entrada es paralelizable (se puede mirar toda la entrada a la vez); generar tokens es secuencial (cada token depende del anterior). Por eso prompt eval rate siempre es mucho más alto que eval rate, en cualquier dispositivo.

---

## ⬜ Pendientes antes de cerrar informe.md completamente

- [ ] Extraer de los logs de turbostat: `PkgWatt` promedio durante cada corrida (restar baseline de reposo ~46-50W)
- [ ] Extraer temperatura CPU (Tctl) promedio/máx durante corridas CPU
- [ ] Si se usó `rocm-smi`, anotar % uso GPU observado (ya tienes notas: máx 99-100% breve, sostenido ~93-95%)
- [ ] Tomar screenshots representativos de terminal (exp1_gpu.png, exp1_cpu.png) si no se hizo ya
- [ ] Rellenar RAM/VRAM usada (o `null` si no se pudo medir, documentando el intento)

---

## 🎯 Siguiente paso: Experimento 2 — Razonamiento Matemático

Mismo flujo que Experimento 1: 3 corridas GPU + 3 corridas CPU, turbostat en paralelo, prompt exacto del enunciado. Diferencia clave a anticipar: la respuesta es mucho más corta (problema matemático, no ensayo), así que `eval count` será bajo y `load duration` pesará proporcionalmente más en el total.
