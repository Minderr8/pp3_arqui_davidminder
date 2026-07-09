INFO128 — Arquitectura de Computadores

## Prueba 3 (modo trabajo): Benchmarking de Inferencia LLM

Dr. Cristóbal Navarro · Universidad Austral de Chile · 2026-S1

## 1. Descripción General

En este proyecto instalarán y ejecutarán un modelo de lenguaje (LLM) multimodal en su computador personal, midiendo el rendimiento de inferencia tanto en CPU como en GPU (si su hardware lo permite). El objetivo es aplicar los conceptos de la Unidad 5 (Arquitecturas Paralelas) a un caso real: contrastar cómo el procesamiento secuencial de la CPU y el procesamiento paralelo de la GPU afectan el rendimiento de una carga de trabajo moderna como la inferencia de redes neuronales.

El proyecto es individual. La instalación de las herramientas en su sistema operativo (Windows, Linux o macOS) es parte del desafío y es responsabilidad de cada estudiante.

## 2. Herramientas

Motor de inferencia: Se recomienda ollama (https://ollama.com). Se instala con un solo comando en Linux/macOS y como aplicación en Windows. Detecta automáticamente GPUs NVIDIA (CUDA), AMD (ROCm) e Intel. Alternativamente pueden usar llama.cpp directamente.

Modelo recomendado: gemma3:4b — es multimodal (texto + imagen), cuantizado, y cabe en 4 GB de VRAM. Si su hardware no soporta este modelo, pueden usar gemma3:1b u otro modelo multimodal liviano justificando la elección.

Monitoreo: Investiguen las herramientas disponibles en su sistema operativo para medir consumo energético (Watts), temperatura y TFLOPS. Ejemplos: nvidia-smi (NVIDIA), rocm-smi (AMD), intel_gpu_top (Intel), powertop, turbostat, sensors (Linux), HWiNFO/HWMonitor (Windows), etc.

## 3. Experimentos

Deben ejecutar 3 experimentos con prompts fijos (no modificables). Cada experimento se ejecuta 3 veces por dispositivo (CPU y GPU por separado) y se reporta el promedio de las 3 ejecuciones. Para forzar ejecución solo en CPU con ollama, usen:

CUDA_VISIBLE_DEVICES="" ollama run ... (Linux/macOS)

En Windows, consulte la documentación de ollama para deshabilitar GPU.

## 3.1 Experimento 1 — Generación de Texto (throughput sostenido)

Este experimento mide la capacidad de generar texto largo de forma sostenida. Usen el siguiente prompt exacto:

"Escribe un ensayo detallado de al menos 500 palabras explicando cómo funciona la jerarquía de memoria en un computador moderno, desde los registros del procesador hasta el almacenamiento en disco. Incluye ejemplos concretos de latencias típicas en cada nivel."

## 3.2 Experimento 2 — Razonamiento Matemático (latencia + cómputo)

Este experimento mide la latencia y carga computacional con una tarea de razonamiento. Usen el siguiente prompt exacto:

"Tres servidores procesan tareas independientemente. El servidor A procesa una tarea en 6 minutos, el servidor B en 10 minutos y el servidor C en 15 minutos. Si los tres trabajan en paralelo sobre un lote de 100 tareas, donde cada tarea se asigna a exactamente un servidor, ¿cómo deben distribuirse las 100 tareas entre los tres servidores para minimizar el tiempo total de procesamiento? Muestra todo el desarrollo matemático paso a paso."


## 3.3 Experimento 3 — Multimodal / Visión (carga mixta)

Este experimento evalúa el procesamiento de imagen + texto. Usen la imagen adjunta (imagen_test.png) y mostrada en Figura 1, proporcionada junto a este enunciado, con el siguiente prompt exacto:

"Describe detalladamente lo que observas en esta imagen. Identifica todos los elementos visibles, incluyendo cualquier ser vivo que pueda estar presente. Explica cómo se relacionan los elementos entre sí en la escena."

*Figura 1: Imagen a testear, la imagen en resolución completa está adjunta en SiveducMD*

## 4. Métricas a Reportar

## 4.1 Métricas Básicas (obligatorias)

Estas métricas son reportadas directamente por ollama/llama.cpp:

- Tokens por segundo (tokens/s)

- Tiempo total de inferencia (segundos)

- Uso de RAM / VRAM (MB o GB)

- Carga de CPU (%) y/o carga de GPU (%)

## 4.2 Métricas Avanzadas (obligatorias — requieren investigación)

Investiguen cómo obtener estas métricas en su sistema operativo y hardware específico. Documenten las herramientas utilizadas y el método de medición:

- TFLOPS — Tera operaciones de punto flotante por segundo

- Watts promedio — Consumo energético promedio durante la inferencia

- Eficiencia energética — GFLOPS/Watt

- Energía total — Watts × tiempo (Joules o Wh)

- Temperatura — mínima, máxima y promedio durante la inferencia

Nota: Si no logran medir alguna métrica avanzada en su hardware específico, documenten qué intentaron, por qué no funcionó, y qué alternativa usarían. La investigación del proceso vale puntos.


## 5. Formato de Entrega

Entreguen un archivo apellido_nombre.zip con la siguiente estructura exacta:

```
apellido_nombre/
├── informe.md
├── metrics.json
├── imagen_test.png
└── screenshots/
├── exp1_cpu.png
├── exp1_gpu.png (si aplica)
├── exp2_cpu.png
├── exp2_gpu.png (si aplica)
├── exp3_cpu.png
├── exp3_gpu.png (si aplica)
└── monitoring.png (herramientas de monitoreo)
```

## 5.1 informe.md

Archivo Markdown con las siguientes secciones obligatorias:

- ## Hardware — Especificaciones completas: modelo de CPU (marca, modelo, núcleos, frecuencia), modelo de GPU (marca, modelo, VRAM), RAM total, sistema operativo.

- ## Modelo — Nombre del modelo utilizado, cuantización, tamaño en disco, justificación si no usaron gemma3:4b.

- ## Metodología — Herramientas usadas para cada métrica, comandos ejecutados, cómo forzaron CPU-only vs GPU.

- ## Resultados — Tablas y/o gráficos con todas las métricas. Los gráficos comparativos CPU vs GPU deben incluir el modelo exacto de la CPU y GPU en el título (ej: 'Tokens/s — Intel i7-12700H vs NVIDIA RTX 3060 6GB').

- ## Análisis — Interpretación de los resultados usando conceptos del curso: paralelismo, SIMD/SIMT, ancho de banda de memoria, Ley de Amdahl, diferencias arquitectónicas CPU vs GPU. Esta sección es la más importante.

- ## Conclusiones — Resumen de hallazgos principales.

## 5.2 metrics.json

Archivo JSON con el siguiente schema exacto:

```
{
"student": "Apellido, Nombre",
"hardware": {
"cpu": {"model": "...", "cores": N, "threads": N, "freq_ghz": N.N},
"gpu": {"model": "...", "vram_gb": N.N},
"ram_gb": N,
"os": "..."
},
"model": {"name": "gemma3:4b", "quantization": "Q4_K_M", "size_gb": N.N},
"experiments": {
"exp1_text": {
"cpu": {
"runs": [
{"tokens_per_sec": N.N, "total_time_sec": N.N,
"ram_mb": N, "cpu_load_pct": N.N,
"tflops": N.N, "watts_avg": N.N,
"gflops_per_watt": N.N, "energy_joules": N.N,
"temp_min_c": N.N, "temp_max_c": N.N, "temp_avg_c": N.N}
],
"average": { ... }
},
"gpu": { "runs": [ ... ], "average": { ... } }
},
"exp2_reasoning": { ... },
"exp3_multimodal": { ... }
}
}
```

Si no tienen GPU o no lograron medir una métrica, usen null como valor. No inventen datos.


## 6. Ejecución en CPU vs GPU

La ejecución en CPU es obligatoria para todos. La ejecución en GPU es obligatoria si su hardware lo soporta, incluyendo GPUs integradas (Intel UHD/Iris, AMD Radeon integrada). Si su laptop no tiene ningún tipo de GPU utilizable por ollama, documenten el intento y trabajen solo con CPU.

## 7. Criterios de Evaluación

| Criterio | Pts Descripción |
| --- | --- |
| Instalación y ejecución | 0.5 Evidencia de ejecución exitosa (screenshots) |
| Métricas básicas | 1.0 Las 4 métricas básicas completas y correctas |
| Métricas avanzadas | 1.0 Investigación y medición de TFLOPS, Watts, eficiencia, energía, temperatura |
| Calidad del metrics.json | 1.0 Schema correcto, datos coherentes con hardware declarado, 3 runs por experimento |
| Análisis e interpretación | 2.0 Uso de conceptos del curso para explicar resultados (SIMD, Amdahl, BW memoria, etc.) |
| Presentación general | 0.5 Informe bien estructurado, gráficos claros con modelo de CPU/GPU en títulos |

Total: 6 puntos + punto base

## 8. Plazo y Entrega

Plazo de entrega: miércoles 15 de julio de 2026, 23:59 hrs. Subir el archivo .zip a la tarea prueba 3 en siveducMD.

## 9. Integridad Académica

Pueden usar inteligencia artificial (IA) para investigar cómo instalar herramientas, entender conceptos, automatizar la ejecución de los experimentos y formatear las salidas al formato requerido (por ejemplo, generar el metrics.json a partir de los logs de ollama). Sin embargo, la sección de Análisis debe reflejar comprensión propia de los conceptos del curso aplicados a sus datos específicos. Copiar análisis genéricos será evidente porque no calzará con el hardware y resultados particulares de cada estudiante. Los datos del metrics.json serán validados automáticamente contra el hardware declarado — datos inventados o copiados serán detectados.
