# INFO128 — Prueba 3: Benchmarking de Inferencia LLM

**Curso:** Arquitectura de Computadores — Universidad Austral de Chile (2026-S1)  
**Docente:** Dr. Cristóbal Navarro  
**Alumno:** David Minder  

Benchmarking de inferencia de un modelo de lenguaje (LLM) local corriendo en CPU y GPU, comparando throughput, latencia y consumo energético en hardware real.

---

## Hardware

| Componente | Especificación |
|---|---|
| CPU | AMD Ryzen 7 5700X — 8 núcleos / 16 hilos, hasta 4.67 GHz |
| GPU | AMD Radeon RX 7700S (Navi 33, RDNA3) |
| RAM | ~31 GB |
| OS | Ubuntu (kernel 6.17.0) |

---

## Modelo

- **Modelo:** `gemma3:4b` (multimodal texto + imagen)
- **Cuantización:** Q4_K_M (default de Ollama)
- **Tamaño en disco:** 2.9 GB
- **Motor de inferencia:** [Ollama](https://ollama.com)

---

## Experimentos

| # | Nombre | Descripción |
|---|---|---|
| 1 | Generación de texto | Ensayo largo — mide throughput sostenido |
| 2 | Razonamiento matemático | Problema de distribución de tareas — mide latencia y cómputo |
| 3 | Multimodal / Visión | Descripción de imagen — mide carga mixta texto + imagen |

Cada experimento se ejecuta **3 veces en CPU** y **3 veces en GPU**, reportando el promedio.

**Forzar ejecución CPU-only:**
```bash
CUDA_VISIBLE_DEVICES="" ollama run gemma3:4b "prompt..."
```

**Ejecución normal (GPU):**
```bash
ollama run gemma3:4b --verbose "prompt..."
```

---

## Métricas recolectadas

**Básicas** (reportadas por Ollama):
- Tokens por segundo (`eval rate`)
- Tiempo total de inferencia
- Uso de RAM / VRAM

**Avanzadas** (herramientas del sistema):
- Watts promedio — `sudo turbostat --interval 2` (CPU) / `sensors` (GPU, campo PPT)
- Temperatura min/max/avg — `sensors` (k10temp para CPU, amdgpu para GPU)
- TFLOPS y eficiencia energética (GFLOPS/Watt)
- Energía total (Joules = Watts × segundos)

**Monitoreo GPU en vivo:**
```bash
watch -n 2 rocm-smi
```

**Log de turbostat:**
```bash
sudo turbostat --interval 2 --out Informe/Screenshots/EXP1/exp1_gpu_run1_turbostat.log
```

---

## Estructura del repositorio

```
pp3_arqui_davidminder/
├── material/
│   ├── INFO128_prueba3-2026.md   # Enunciado oficial
│   ├── INFO128_prueba3-2026.pdf
│   └── imagen_test.png           # Imagen para experimento 3
├── Modelfile/
│   └── Modelfile-cpu             # Modelfile con num_gpu=0 para forzar CPU
├── Avances/                      # Bitácora día a día
│   ├── 09_jul_13pm.txt
│   ├── 10_jul.txt
│   ├── 11_jul
│   └── 12_jul.txt
├── Informe/
│   ├── informe.md                # Informe final (entregable)
│   ├── metrics.json              # Métricas en formato JSON (entregable)
│   ├── Reseumenes/               # Resúmenes de experimentos
│   └── Screenshots/              # Logs y capturas por experimento
│       ├── EXP1/
│       └── EXP2/
└── README.md
```

---

## Herramientas utilizadas

| Herramienta | Uso |
|---|---|
| `ollama` | Motor de inferencia, carga y ejecuta el modelo |
| `turbostat` | Consumo (PkgWatt) y frecuencia del CPU |
| `sensors` (lm-sensors) | Temperatura y watts (PPT) de CPU y GPU |
| `rocm-smi` | Monitoreo en vivo de la GPU AMD |

---

## Plazo de entrega

Miércoles 15 de julio de 2026, 23:59 hrs — entrega en SiveducMD (`minder_david.zip`).
