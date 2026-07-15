# Resumen Consolidado — Benchmarking LLM (gemma3:4b, GPU vs CPU)
### Todo lo medido y calculado, listo para escribir el informe

**Hardware:** AMD Ryzen 7 5700X (8c/16t) + AMD Radeon RX 7700S (RDNA3, 8 GB VRAM), 31 GB RAM, Ubuntu, Ollama + ROCm.
**Modelo:** gemma3:4b (GPU) / gemma3-cpu derivado con `num_gpu 0` (CPU).

---

## ✅ Estado: TODO MEDIDO Y CALCULADO

Los 3 experimentos completos, 3 corridas GPU + 3 CPU cada uno, con turbostat + sensors en paralelo. Ya no queda nada pendiente de medir. Solo falta vaciar en `metrics.json` (hecho, ver archivo aparte) y escribir el `informe.md`.

**Nota sobre tokens/s:** se usan los valores de la tanda del 14-jul (Exp1, Exp2) y 13-jul (Exp3), porque coinciden temporalmente con las mediciones de watts/temp. Así todas las métricas provienen de la misma tanda de corridas.

---

## Tabla maestra de datos (promedios de 3 corridas)

| Métrica | Exp1 GPU | Exp1 CPU | Exp2 GPU | Exp2 CPU | Exp3 GPU | Exp3 CPU |
|---|---|---|---|---|---|---|
| eval rate (tok/s) | 63.21 | 10.94 | 63.64 | 11.35 | 63.53 | 11.02 |
| tiempo total (s) | 21.52 | 109.68 | 21.37 | 122.15 | 12.44 | 76.56 |
| eval count (tokens) | 1246 | 1175 | 1231 | 1362 | 635 | 741 |
| GPU PPT (W) | 96.5 | 6.1 | 87.3 | 5.1 | 61.2 | 5.2 |
| GPU junction (°C) avg | 77.1 | reposo | 73.5 | reposo | 67.6 | reposo |
| GPU junction (°C) max | 98 | — | 98 | — | 96 | — |
| CPU PkgWatt (W) | 59.5 | 72.0 | 39.8 | 65.5 | 58.9 | 71.5 |
| CPU Busy% | 24.7 | 63.2 | 10.7 | 48.8 | 22.4 | 60.2 |

**Speedup GPU/CPU (tok/s):** Exp1 = 5.78x · Exp2 = 5.61x · Exp3 = 5.76x → **~5.7x constante**

*(GPU PPT ~5-6W y "reposo" en columnas CPU confirman que la GPU no participó en las corridas CPU.)*

---

## Cálculos derivados

### TFLOPS efectivos (fórmula: 2 × N_params × tok/s, N=4e9)
- **GPU:** 2 × 4e9 × 63.5 = **0.508 TFLOP/s**
- **CPU:** 2 × 4e9 × 11.0 = **0.088 TFLOP/s**

### Utilización vs peak teórico RX 7700S (FP16)
- Peak vectorial: 20.5 TFLOPS → utilización GPU = **2.48%**
- Peak matricial: 41 TFLOPS → utilización GPU = **1.24%**

### Eficiencia energética (tok/s ÷ watts)
- **GPU:** 63.5 / 92 = **0.69 tok/s/W**
- **CPU:** 11.0 / 70 = **0.157 tok/s/W**
- GPU ~4.4x más eficiente

### Energía total por tarea (watts × tiempo total)
- **GPU:** 92 × 21 = **1932 J**
- **CPU:** 70 × 110 = **7700 J**
- GPU ~4x menos energía por respuesta

### RAM / VRAM
- VRAM modelo (ollama ps): **2.9 GB**
- VRAM total en uso (rocm-smi): **4.97 GB** (incluye escritorio gráfico)
- RAM modelo en CPU (free -h, diferencia usado): **~3.5 GB**
- VRAM total GPU: 8.57 GB

---

## Hallazgos para la sección de Análisis (TODOS tuyos, con datos propios)

1. **Paradoja cómputo vs memoria (hallazgo principal):** la GPU usa solo ~2.5% de su capacidad de cálculo (0.508 de 20.5 TFLOPS) y aun así es 5.7x más rápida. → El cuello de botella NO es cómputo, es **ancho de banda de memoria**: mover ~2.9 GB de pesos desde VRAM en cada token. La GPU gana por su mayor bandwidth (VRAM), no por sus núcleos.

2. **Speedup constante ~5.7x** en los 3 experimentos pese a tareas muy distintas (ensayo / matemática / visión). → Refleja una constante de hardware (relación de bandwidth VRAM vs RAM), no algo dependiente del tipo de tarea. El eval rate mide el costo de generar cada token (una pasada por la red = misma operación estructural sin importar el contenido).

3. **Eficiencia energética invertida a la intuición:** la GPU consume más watts instantáneos (92 vs 70) pero es 4.4x más eficiente en tok/s/W, y gasta 4x menos energía total por tarea. La CPU pierde por partida doble: menos eficiente por segundo Y tarda 5x más.

4. **Varianza de tokens por especificidad del prompt (Exp2):** el prompt sin límite explícito ("muestra todo el desarrollo") produjo ~40% de spread en eval count (958-1607), vs <1% en Exp1 ("al menos 500 palabras"). → El determinismo del output depende de la especificidad de la instrucción.

5. **Page cache / jerarquía de memoria (Exp3):** la corrida 1 fue mucho más lenta en load+prompt eval que las corridas 2-3, en AMBOS dispositivos. → Efecto del page cache del kernel: la primera lectura toca disco (NVMe), las siguientes salen de RAM. Verificable con `drop_caches`.

6. **Costo de la imagen en tokens (Exp3):** prompt eval count saltó a ~321-326 tokens (vs 108 en Exp2). Una imagen en gemma3 ≈ 256 tokens de contexto fijos. → Explica por qué el prompt eval del experimento multimodal es mucho más pesado.

7. **Ley de Amdahl (Exp1-2):** durante corridas GPU, turbostat mostró un solo core lógico saltando a ~90-100% mientras el resto ocioso. → Es la parte secuencial (tokenización/sampling) que no se paraleliza, coexistiendo con la parte paralela (matmul en GPU).

---

## ⚠️ Notas de rigor a incluir en el informe (suman puntos)

- Los TFLOPS y la energía son **estimaciones de orden de magnitud**: N=4e9 redondeado, watts promedio de rangos, tiempos aproximados. Suficiente para las conclusiones cualitativas (limitado por memoria, gana la GPU), no son medidas exactas.
- Anomalía de idioma (Exp2): una corrida CPU respondió en inglés pese al prompt en español; se descartó y repitió. Fuente de no-determinismo del modelo (sampling), no del hardware.
- Discrepancia menor: prompt eval count Exp3 difiere 5 tokens entre GPU (321) y CPU (326) por espaciado distinto del texto del prompt entre comandos. Documentar honestamente.

---

## Estructura sugerida del informe.md (para mañana)

1. **Portada / intro:** objetivo, hardware exacto (con modelos), modelo LLM.
2. **Metodología:** cómo se forzó CPU (Modelfile num_gpu 0), herramientas (turbostat, sensors, rocm-smi), cómo se midió cada métrica. Incluir el gap de sensors ya resuelto.
3. **Resultados:** la tabla maestra + gráficos (tok/s por experimento, watts, energía por tarea). Los gráficos DEBEN llevar el modelo de CPU/GPU en el título (lo exige el enunciado).
4. **Análisis:** los 7 hallazgos de arriba, con tus palabras. Este es el corazón (2.0 pts).
5. **Conclusiones:** GPU gana en velocidad, eficiencia y energía; el porqué es el ancho de banda de memoria; la inferencia batch=1 está memory-bound.
6. **Limitaciones:** las notas de rigor de arriba.
