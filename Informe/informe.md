# Informe — Benchmarking de Inferencia LLM: CPU vs GPU
**Prueba 3 — Arquitectura de Computadores (INFO128)**
**Estudiante:** David Minder

---

## Hardware

- **CPU:** AMD Ryzen 7 5700X — 8 núcleos / 16 hilos, frecuencia base 3.4 GHz (boost hasta ~4.6 GHz).
- **GPU:** AMD Radeon RX 7700S (arquitectura RDNA3 / Navi 33), 8 GB VRAM (GDDR6).
- **RAM:** 31 GB (utilizables).
- **Placa madre:** ASRock B550 Phantom Gaming 4.
- **Sistema operativo:** Ubuntu (kernel 6.17).
- **Stack GPU:** ROCm (la GPU fue detectada automáticamente por Ollama, sin necesidad de `HSA_OVERRIDE_GFX_VERSION`).

---a

## Modelo

- **Nombre:** gemma3:4b (Google Gemma 3, 4 mil millones de parámetros).
- **Cuantización:** Q4_K_M.  
- **Tamaño en disco:** ~2.9 GB (cuantizado).
- **Tamaño en memoria durante inferencia:** ~2.9 GB en VRAM (GPU) / ~3.5 GB en RAM (CPU).
- Se utilizó el modelo solicitado por el enunciado sin sustituciones. Para forzar ejecución CPU-only se creó un modelo derivado `gemma3-cpu` (ver Metodología).

---

## Metodología
primero cree un repo en github y subi el material https://github.com/Minderr8/pp3_arqui_davidminder, me descarge Ollama con el comando: curl -fsSL `https://ollama.com/install.sh | sh` , revise mi GPU con: `lspci | grep -i vga`, mi RAM con: `free -h`, mi CPU con: `lscpu`, luego descarge el modelo de IA gemma3:4b con Ollama, el cual descarga los pesos (3-4GB) y los guarda en -/.ollama/models dejandolo listo para correr.

para confirmar la instalacion de del modelo aplique el comando: `ollama list`, luego para verificar si Ollama detecta mi GPU sin hacer nada en una terminal uso el modelo con: `ollama run gemma3:4n "hola"` y en otra terminal en paralelo con: `ollama ps`, y confirmo que ollama detecta mi GPU sin problemas

### Modelos y modo de ejecución

- **GPU:** `ollama run gemma3:4b --verbose "<prompt>"`
- **CPU:** `ollama run gemma3-cpu --verbose "<prompt>"`

**Cómo se forzó CPU-only:** las variables de entorno (`CUDA_VISIBLE_DEVICES`, `ROCR_VISIBLE_DEVICES`, `HIP_VISIBLE_DEVICES`) NO funcionaron, porque Ollama corre como servicio systemd persistente (`ollama serve`) que no re-lee las variables de la shell. La solución fue crear un modelo derivado mediante un Modelfile con el parámetro `PARAMETER num_gpu 0`:

```
ollama create gemma3-cpu -f Modelfile-cpu
```

Se verificó con `ollama ps` que mostrara `100% CPU` antes de cada corrida CPU (y `100% GPU` en las corridas GPU).

### Herramientas de medición (una por familia de métrica)

| Métrica | Herramienta | Detalle |
|---|---|---|
| Tokens/s, tiempo, tokens generados | `ollama --verbose` | Campos `eval rate`, `total duration`, `eval count` |
| Watts CPU (package) + carga por core | `turbostat --interval 2` | Columnas `PkgWatt` y `Busy%` |
| Watts GPU (PPT) + temperaturas | `sensors` (driver amdgpu) | Campo `PPT`, `junction`, `edge`, `mem`; CPU vía `k10temp` (`Tctl`) |
| Utilización GPU en vivo | `rocm-smi` | Monitoreo visual durante corridas |
| VRAM | `rocm-smi --showmeminfo vram` + `ollama ps` | Memoria usada del driver y del modelo |
| RAM | `free -h` | Diferencia de columna `usado` antes/después de cargar el modelo |

Cada experimento se ejecutó **3 veces por dispositivo** (3 GPU + 3 CPU) con `turbostat` y `sensors` corriendo en paralelo en terminales separadas. Los watts y temperaturas se promediaron sobre todos los intervalos de 2 s de cada corrida.

### Prompts utilizados

- **Experimento 1 (texto largo):** ensayo de mínimo 500 palabras sobre la jerarquía de memoria.
- **Experimento 2 (razonamiento matemático):** distribución óptima de 100 tareas entre 3 servidores, con desarrollo paso a paso.
- **Experimento 3 (multimodal/visión):** descripción detallada de la imagen `imagen_test.png`.

### Cálculos derivados

- **TFLOPS efectivos:** `2 × N_params × (tokens/s)`, con N = 4×10⁹.
- **Eficiencia energética:** `GFLOPS efectivos / watts`.
- **Energía por tarea:** `watts × tiempo_total`.
- **Utilización:** `TFLOPS efectivos / TFLOPS peak teórico` (RX 7700S: 20.5 vectorial / 41 matricial FP16).

> **Nota de rigor:** los TFLOPS y la energía son estimaciones de orden de magnitud (N redondeado, watts promediados, tiempos aproximados). Son suficientes para las conclusiones cualitativas, no pretenden ser medidas exactas.

---

## Resultados

### Tabla comparativa (promedios de 3 corridas)

| Métrica | Exp1 GPU | Exp1 CPU | Exp2 GPU | Exp2 CPU | Exp3 GPU | Exp3 CPU |
|---|---|---|---|---|---|---|
| Tokens/s | 63.21 | 10.94 | 63.64 | 11.35 | 63.53 | 11.02 |
| Tiempo total (s) | 21.52 | 109.68 | 21.37 | 122.15 | 12.44 | 76.56 |
| Tokens generados | 1246 | 1175 | 1231 | 1362 | 635 | 741 |
| Watts GPU (PPT) | 96.5 | 6.1* | 87.3 | 5.1* | 61.2 | 5.2* |
| Watts CPU (Pkg) | 59.5 | 72.0 | 39.8 | 65.5 | 58.9 | 71.5 |
| Temp GPU junction avg (°C) | 77.1 | — | 73.5 | — | 67.6 | — |
| Temp GPU max (°C) | 98 | — | 98 | — | 96 | — |
| Carga CPU (Busy%) | 24.7 | 63.2 | 10.7 | 48.8 | 22.4 | 60.2 |

*\*GPU en reposo durante corridas CPU (confirma que no participó).*

### Métricas derivadas

| | GPU | CPU |
|---|---|---|
| TFLOPS efectivos | 0.508 | 0.088 |
| Utilización vs peak vectorial (20.5) | 2.48% | — |
| Utilización vs peak matricial (41) | 1.24% | — |
| Eficiencia (tok/s por watt) | 0.69 | 0.157 |
| Energía por tarea Exp1 (J) | ~1932 | ~7700 |

### Speedup GPU vs CPU (tokens/s)
- Exp1: 5.78x · Exp2: 5.61x · Exp3: 5.76x → **~5.7x constante en las 3 tareas.**

### Memoria
- VRAM ocupada por el modelo (`ollama ps`): **2.9 GB**
- VRAM total en uso (`rocm-smi`, incluye escritorio): **4.97 GB** de 8.57 GB
- RAM ocupada por el modelo en CPU (`free -h`): **~3.5 GB**

> **[INSERTAR GRÁFICOS AQUÍ]** — como mínimo: (1) Tokens/s CPU vs GPU por experimento, (2) Energía por tarea, (3) Watts. Título con modelo exacto, ej: *"Tokens/s — AMD Ryzen 7 5700X vs AMD Radeon RX 7700S"*.

---

## Análisis

> Esta es la sección más importante (2.0 pts). Debe estar escrita con TUS palabras e interpretar los datos usando conceptos del curso. Abajo tienes el andamiaje: cada punto es un hallazgo real tuyo con la pregunta que debes responder. Desarrolla cada uno en 1-2 párrafos.

### 1. La paradoja cómputo vs. memoria (hallazgo central)
*Datos:* la GPU solo alcanza ~2.5% de su capacidad de cómputo teórica (0.508 de 20.5 TFLOPS), y aun así es ~5.7x más rápida que la CPU.
*Responde:* si a la GPU le sobra el 97% de su poder de cálculo, ¿por qué no va aún más rápido? ¿Qué la limita? Conecta con el concepto de **ancho de banda de memoria**: en inferencia batch=1 hay que leer los ~2.9 GB de pesos desde la VRAM en cada token generado. ¿Por qué eso favorece a la GPU (VRAM ~hundreds GB/s) sobre la CPU (RAM ~decenas GB/s)? → La inferencia está *memory-bound*, no *compute-bound*.

### 2. Speedup constante independiente de la tarea
*Datos:* ~5.7x en las 3 tareas (ensayo, matemática, visión), pese a ser cognitivamente muy distintas.
*Responde:* ¿por qué el tipo de contenido no cambia el speedup? Conecta con qué mide realmente el `eval rate`: el costo de generar UN token = una pasada por la red (multiplicación de matrices), que es la misma operación estructural sin importar si el token es una ecuación o una palabra. → El speedup refleja una relación de hardware (bandwidth VRAM/RAM), no la semántica de la tarea.

### 3. Diferencias arquitectónicas CPU vs GPU (SIMT vs SIMD)
*Responde:* explica por qué una GPU es mejor para esto. Conecta con **SIMT** (miles de hilos ejecutando la misma operación sobre distintos datos — ideal para matmul) vs la CPU con pocos núcleos y **SIMD** de ancho limitado (AVX). ¿Cómo se relaciona esto con que la GPU tenga tantas ALU y mayor bandwidth? Usa tus datos de utilización para matizar: aunque la GPU es SIMT-masiva, en batch=1 ni siquiera satura sus ALU (2.5%), lo que refuerza que el límite es alimentar datos, no calcular.

### 4. Ley de Amdahl en la práctica
*Datos:* durante las corridas GPU, `turbostat` mostró UN core lógico saltando a ~90-100% mientras el resto quedaba ocioso (Busy% global bajo, ~10-25%).
*Responde:* ¿qué parte del proceso es secuencial y no se paraleliza en la GPU? (tokenización, sampling, orquestación en CPU). Conecta con **Amdahl**: aunque la parte pesada (matmul) se acelera masivamente en GPU, la fracción secuencial en CPU pone un techo. ¿Se ve ese "core solitario" como evidencia?

### 5. Eficiencia energética contraintuitiva
*Datos:* la GPU consume más watts instantáneos (92 vs 70) pero es 4.4x más eficiente (0.69 vs 0.157 tok/s/W) y gasta 4x menos energía total por tarea (1932 vs 7700 J).
*Responde:* ¿cómo puede ser más eficiente algo que "gasta más"? ¿Por qué la CPU pierde por partida doble (menos tok/s/W Y más tiempo acumulando consumo)? Distingue eficiencia instantánea (tok/s/W) de energía total (J).

### 6. Determinismo del output y especificidad del prompt (Exp2)
*Datos:* el prompt sin límite de longitud produjo ~40% de variación en tokens generados entre corridas (958-1607), vs <1% en Exp1 (que pedía "mínimo 500 palabras").
*Responde:* ¿por qué una instrucción cualitativa genera más varianza que una cuantitativa? Relevancia para diseñar benchmarks reproducibles.

### 7. Jerarquía de memoria observable (Exp3, page cache)
*Datos:* en Exp3 la corrida 1 fue notablemente más lenta en `load`+`prompt eval` que las corridas 2-3, en AMBOS dispositivos.
*Responde:* ¿por qué la primera es más lenta? Conecta con el **page cache** del kernel: la 1ª lectura toca disco (NVMe), las siguientes salen de RAM. Es la jerarquía de memoria (disco→RAM→caché) visible en la práctica. (Opcional: mencionar que se puede verificar con `drop_caches`.)

### 8. Costo de la imagen en tokens (Exp3, opcional)
*Datos:* el `prompt eval count` saltó a ~321-326 tokens en Exp3 vs 108 en Exp2. Una imagen en gemma3 ≈ 256 tokens de contexto.
*Responde:* qué implica para el costo de procesar entradas multimodales.

---

## Conclusiones

> Resume en 4-6 puntos los hallazgos principales. Andamiaje:

1. La GPU supera a la CPU por ~5.7x en velocidad de generación, de forma consistente en las 3 tareas.
2. El cuello de botella real de la inferencia batch=1 es el **ancho de banda de memoria**, no la capacidad de cómputo (la GPU usa solo ~2.5% de sus TFLOPS).
3. La GPU es además más eficiente energéticamente: [completa con tus números — 4.4x en tok/s/W y 4x en energía total].
4. La ventaja de la GPU proviene de su arquitectura [SIMT / masivamente paralela / mayor bandwidth], no del tipo de tarea.
5. [Tu conclusión sobre Amdahl / la fracción secuencial que aún corre en CPU].
6. [Cierre personal: qué aprendiste sobre por qué el hardware especializado gana].