# hpl-apple-m4

Evaluación de rendimiento con **HPL (High Performance Linpack)** sobre un MacBook Air con chip Apple M4 (16 GB RAM, macOS Sequoia). Este repositorio contiene el código fuente de HPL, el Makefile configurado, los resultados de cada fase de prueba y el informe completo.

## Requisitos

- macOS (Apple Silicon: M1/M2/M3/M4)
- [Homebrew](https://brew.sh/)


## 1. Instalar dependencias

```bash
brew install gcc open-mpi
```

Esto instala GCC (usado por `mpifort` para el enlazador Fortran) y OpenMPI. El compilador C (`mpicc`) usará Apple Clang automáticamente, optimizado para ARM64.

## 2. Clonar este repositorio

```bash
git clone https://github.com/GabrielaHR10/hpl-apple-m4.git
cd hpl-apple-m4
```

Todo el proyecto vive dentro de este repositorio:

- `hpl-2.3/` — código fuente de HPL 2.3 con `Make.MacSilicon` ya configurado
- `datResultados/` — `HPL.dat` y logs `.log` de cada fase de prueba
- `markdownInforme/` — informe convertido a Markdown + imágenes
- `entrega/` — archivos de entrega
- `InformeHPLmfqcYA.pdf` — informe en PDF

El `.gitignore` excluye solo archivos generados al compilar (binarios, `.o`, `.a`), así que después de clonar solo falta compilar.

## 3. Revisar el Makefile (`hpl-2.3/Make.MacSilicon`)

```bash
cd hpl-2.3
```

Verifica estas variables si tu ruta de clonado o instalación de Homebrew es distinta a la original:

| Variable | Valor esperado | Motivo |
|---|---|---|
| `ARCH` | `MacSilicon` | Debe coincidir con el sufijo del archivo `Make.<ARCH>` |
| `TOPdir` | Ruta donde clonaste el repo + `/hpl-2.3` | **Ajústala con TU dirección donde clonaste el repo** |
| `MPdir` / `MPinc` / `MPlib` | `/opt/homebrew/...` | Ruta de OpenMPI instalado vía Homebrew — ajusta si tu Homebrew está en `/usr/local` (Mac Intel) |
| `LAlib` | `-framework Accelerate` | Usa Apple Accelerate (BLAS optimizado con AMX) en vez de una BLAS genérica |
| `CC` | `mpicc` | Compilador C (Apple Clang vía OpenMPI) |
| `LINKER` | `mpifort` | Necesario porque Accelerate tiene funciones en Fortran |

## 4. Compilar

```bash
make arch=MacSilicon
```

Si compila correctamente, el ejecutable queda en:

```
bin/MacSilicon/xhpl
```

(Esta carpeta no viene en el repo — está en `.gitignore` porque se regenera al compilar.)

## 5. Usar los `HPL.dat` ya incluidos en el repo

Cada fase de prueba documentada en `InformeHPL.md` tiene su `HPL.dat` correspondiente guardado en `datResultados/` (un nivel arriba de `hpl-2.3/`). Para replicar una fase específica, copia el `HPL.dat` de esa fase a `bin/MacSilicon/HPL.dat`:

```bash
cp ../datResultados/HPL_fase5.dat bin/MacSilicon/HPL.dat
```

Los tres parámetros principales que cambian entre pruebas son:

- **N**: tamaño de la matriz (mayor N = más memoria y más tiempo, pero potencialmente más GFLOPS).
- **NB**: tamaño de bloque, afecta el uso de caché.
- **P × Q**: grilla de procesos MPI, donde `P × Q` debe ser igual al número de procesos (`np`) con el que se ejecuta `mpirun`.

Si quieres probar una configuración nueva, edita `HPL.dat` manualmente manteniendo los parámetros algorítmicos fijos usados en todo el informe: PFACT=2, RFACT=1, BCAST=1, DEPTH=1, NBMIN=4, NDIV=2, SWAP=2, EQUIL=1.

## 6. Ejecutar

Desde `hpl-2.3/bin/MacSilicon/`:

```bash
cd bin/MacSilicon
mpirun -np <np> ./xhpl | tee ../../../datResultados/<nombre_de_corrida>.log
```

Donde `<np>` debe coincidir con `P × Q` definido en `HPL.dat`.

### Secuencia de pruebas replicada en el informe

| Fase | Objetivo | np | Parámetros clave |
|---|---|---|---|
| 1 | Verificación | 8 | N=5000, NB=192, P=2, Q=4 |
| 2 | Efecto de N | 8 | N=10000/20000/30000, NB=192, P=2, Q=4 |
| 3 | Efecto de NB | 8 | N=20000, NB=128/192/256, P=2, Q=4 |
| 4 | Efecto de P×Q y np | 6/8/10 | N=20000, NB=256, distintas grillas |
| 5 | Corrida final / N óptimo | 8 | NB=256, P=1, Q=8, N entre 30000–40192 |

Repite este orden si quieres reproducir el proceso completo de optimización, o salta directo a la Fase 5 con los parámetros óptimos ya encontrados (`N=37500, NB=256, P=1, Q=8, np=8`) si solo quieres el mejor resultado.

## 7. Verificar recursos del sistema (opcional, para comparar contra tu propio hardware)

```bash
sysctl -n hw.logicalcpu
sysctl -n hw.physicalcpu
system_profiler SPHardwareDataType
sudo powermetrics -n 1 --samplers cpu_power | grep -E "E-Cluster|P-Cluster|frequency"
```

Esto permite recalcular el rendimiento teórico máximo para tu propio chip, ya que las frecuencias de los núcleos P y E varían entre variantes de Apple Silicon.

