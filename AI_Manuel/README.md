# AI_Manuel

Tres líneas de trabajo independientes:

1. **Pipeline de datos GAMS/OPF** (raíz de esta carpeta) — genera datasets sintéticos de series temporales para una red de distribución de 14 buses (365 días × ~289 intervalos de 5 minutos), ejecuta modelos de optimización GAMS y agrega los resultados en CSV para entrenamiento de ML.

2. **Herramientas de cumplimiento NTS** (`NTS/`) — scripts y modelos de pequeña señal para la norma de supervisión de la red española (NTS 631): conversión PDF→Markdown de documentos regulatorios, y análisis de autovalores del sistema de referencia de dos máquinas de NTS §5.10.2.1.

3. **Modelos de convertidor grid-forming VSG** (`VSG/`) — modelos DAE simbólicos independientes (control externo PI-VSG + lazos internos de corriente/tensión) para un convertidor grid-forming, basados en Matas-Díaz et al. (JMPSCE 2025). Cada script imprime sus ecuaciones y genera un PDF de documentación.

`pydae_env/` (raíz del repo, junto a `AI_Manuel/`) es un virtualenv de Python commiteado — no es código del proyecto.

## Ejecutar los scripts

Todos los scripts son independientes. Ejecutar desde la raíz del repo:

```powershell
python AI_Manuel/pdf_to_markdown.py <archivo.pdf> [salida.md]   # PDF → Markdown, imágenes embebidas en base64
python AI_Manuel/creacarpetas_v1.py                              # Crea la estructura de carpetas
python AI_Manuel/datosrandom_v1.py                               # Genera datos sintéticos
python AI_Manuel/llamagams_v1.py                                 # Ejecuta el batch de GAMS (versión actual — llamagams.py es una copia antigua, no usar)
python AI_Manuel/datos_entrada_v1.py                              # Construye el CSV de entrada
python AI_Manuel/datos_salida_v1.py                               # Construye el CSV de salida
python AI_Manuel/VSG/pi_vsg_cc_dae.py                             # Imprime las ecuaciones DAE y genera pi_vsg_cc_model.pdf (también pi_cc_dae.py, pi_vsg_vc_dae.py)
```

Python está instalado desde Microsoft Store (`python` o `python -m pip`).

## Arquitectura del pipeline GAMS

**Flujo de datos (debe ejecutarse en este orden):**

```
creacarpetas_v1.py
  → data_set/day_N/minutoNNNN/   (364 días × ~289 pasos de tiempo)

datosrandom_v1.py
  → lee las plantillas de day_1/ (LOAD_H_DATA.txt, LOAD_I_DATA.txt, GEN_DATA.txt)
  → escribe versiones perturbadas en day_2 … day_365

llamagams_v1.py
  → ejecuta conv_data.gms (guarda archivos de reinicio .g00)
  → ejecuta MinPerd_Base_perdidas.gms (escribe RESULT_B2B_minNNNN.put)

datos_entrada_v1.py  →  inDATA/prueba.txt   (28 columnas: P/Q por bus, carga neta - generación)
datos_salida_v1.py   →  outDATA/outset.txt  (28 columnas: V por bus + flujos de línea + pérdidas)
```

**Red:** 14 buses, Sb = 10 MVA. Pasos de tiempo: minutos 3, 8, 13, … 1443 (paso 5). Los archivos `.gms` del modelo están en `data_set/day_1/`, junto con las plantillas del día 1.

**Archivos clave por paso de tiempo:**
- `LOAD_H_DATA.txt` — P/Q residencial en buses 1,3,4,5,6,8,10,11,12,14
- `LOAD_I_DATA.txt` — P/Q industrial en buses 1,3,7,9,10,12,13,14
- `GEN_DATA.txt` — 15 generadores (buses 3–11), columnas: Name, Bus, P, Pmax, Pmin
- `RESULT_B2B_minNNNN.put` — salida de GAMS: tensiones de bus (líneas 10–23, 30–42) + pérdidas (línea 0)

`OPF/` contiene una copia anterior/de trabajo del mismo proyecto GAMS (incluye artefactos `.log`/`.lst` commiteados) — `data_set/` es el pipeline actual, se prefiere ese.

## Herramientas NTS

### pdf_to_markdown.py

Convierte un PDF a Markdown usando `pymupdf4llm`, embebiendo cada figura extraída como base64 inline (sin carpeta `_fig/` separada — eso era el comportamiento de una versión anterior del script, ubicada antes en `NTS/`). Usa siempre rutas absolutas internamente para evitar problemas de directorio de trabajo al estilo GAMS.

```python
pdf_to_markdown(pdf_path, output_path=None)
```

### Sistema de referencia de dos máquinas NTS §5.10.2.1 (análisis de autovalores)

Existen dos implementaciones paralelas del mismo benchmark, ambas usando [pydae](https://github.com/jmmauricio/pydae):

- **`NTS/NTS_emo/`** — backend por defecto de `pydae` (requiere compilador C). Instalar con `pip install -e ".[dev]"`; tests con `pytest` (testpaths = `tests/`). Puntos de entrada: `build_nts_model()`, `xl_sweep()`, `plot_eigenvalue_sweep()`.
- **`NTS/NTS_emo_Juan/`** — backend CasADi, **sin necesidad de compilador C** (usar esta en máquinas sin herramientas de compilación, p. ej. instalaciones de Python de Microsoft Store). Instalar con `pip install -e .`; tests con `pytest tests/ -v -m "not slow"` (rápidos) o `pytest tests/ -v` (completos, incluye el barrido de XL). Puntos de entrada: `build_model()`, `xl_sweep()`, `plot_eigenvalue_sweep()`.

Ambas barren la reactancia de línea XL de 0.01 a 0.6 pu en un sistema de 4 buses y 100 MVA (2 generadores síncronos, excitatrices ST4B+PSS2A y ST1+IEEEG1) y grafican los modos electromecánicos en el plano complejo. Criterio de aceptación (NTS §5.10.3.1): todos los modos en [0.1, 1.5] Hz deben tener amortiguamiento ≥ 5% en todo el barrido.

`NTS/nts/cases/base/` es un caso de benchmark pydae relacionado pero independiente (equivalente de 5 buses de REE, validación por respuesta al escalón frente a una curva de referencia NTS) — ver su propio `main.py` (patrón `build()`/`ini()`/`run()`) y README.

### Corpus documental

| Archivo | Documento fuente |
|------|-----------------|
| `NormaTecnicaSupervision631_v2_publicada.md` | REE NTS 631 v2 (2020) — norma de supervisión de conformidad de la TSO española para módulos de generación eléctrica bajo el Reglamento UE 2016/631 |
| `4215-2016.md` | IEEE Std 421.5-2016 — modelos de sistemas de excitación para estudios de estabilidad de sistemas de potencia |

Cada documento tiene un directorio `<stem>_fig/` correspondiente con figuras PNG, de cuando `pdf_to_markdown.py` escribía las figuras a disco en vez de embeberlas inline (ver arriba). Está pendiente una conversión de Kundur (*Power System Stability and Control*) — los intentos previos con `marker` fueron demasiado lentos sin GPU/CUDA.

### Conceptos clave del dominio

- **MGE** (Módulo de Generación de Electricidad) — el módulo de generación de electricidad que se certifica bajo NTS 631
- **PEC** — procedimiento de evaluación de conformidad (por Certificado / por Prueba / por Simulación)
- **CAMGE** — módulo de servicios auxiliares (STATCOM, PPC, compensador síncrono, BESS)
- **NTS §5.10.2.1** — el sistema de prueba de dos máquinas que implementan `NTS_emo` / `NTS_emo_Juan`

## Modelos de convertidor grid-forming VSG

`VSG/` contiene scripts de modelos DAE simbólicos independientes que implementan la arquitectura de control de Matas-Díaz et al., "A Systematic Small-signal…" (JMPSCE 2025):

- `pi_cc_dae.py` — solo controlador de corriente PI (CC-ICL), §II.C.2, ecs. (17)-(20): 4 estados, 2 variables algebraicas.
- `pi_vsg_cc_dae.py` — modelo completo con lazo interno controlado por corriente (filtro LCL + PI-VSG OCL + CC-ICL), §II.A-C.2: 13 estados, 8 variables algebraicas.
- `pi_vsg_vc_dae.py` — modelo completo con lazo interno controlado por tensión (filtro LCL + PI-VSG OCL + VC-ICL, PI en cascada), §II.A/B/C.1: 13 estados, 8 variables algebraicas.

Cada script define `dx/dt = f(x, y, u)` y `0 = g(x, y, u)` simbólicamente, imprime las ecuaciones por stdout y genera un PDF de documentación `<name>_model.pdf` al ejecutarse directamente.

## Dependencias

```
numpy, scipy, sympy, matplotlib, casadi   # análisis de pequeña señal / DAE (VSG, NTS_emo*)
pymupdf4llm                               # conversión de PDF
pandas                                     # pipeline de datos
pydae, cffi, hatchling, pytest             # solo paquetes NTS_emo / NTS_emo_Juan
```

Instalación: `pip install <paquete>` o `python -m pip install <paquete>` si `pip` no está en el PATH.
