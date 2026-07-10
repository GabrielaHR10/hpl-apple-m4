# hpl-apple-m4

Evaluación de rendimiento con **HPL (High Performance Linpack)** sobre un MacBook Air con chip Apple M4 (16 GB RAM, macOS Sequoia). Este repositorio contiene la configuración, los scripts de ejecución y los resultados usados en el informe `InformeHPL.md`.

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

Este repositorio **no incluye el código fuente de HPL** (se puede volver a descargar libremente de Netlib), solo contiene el Makefile configurado (`Make.MacSilicon`), los `HPL.dat`/logs de cada fase de prueba en `datResultados/` y el informe.

## 3. Descargar y descomprimir HPL 2.3

```bash
mkdir -p ~/hpl-proyecto
cd ~/hpl-proyecto
curl -O https://www.netlib.org/benchmark/hpl/hpl-2.3.tar.gz
tar -xzf hpl-2.3.tar.gz
```

## 4. Copiar el Makefile del repo al código fuente de HPL

```bash
cp ~/hpl-apple-m4/Make.MacSilicon ~/hpl-proyecto/hpl-2.3/
cd ~/hpl-proyecto/hpl-2.3
```

Revisa estas variables si tu ruta de instalación es distinta a la original:

| Variable | Valor esperado | Motivo |
|---|---|---|
| `ARCH` | `MacSilicon` | Debe coincidir con el sufijo del archivo `Make.<ARCH>` |
| `TOPdir` | `$(HOME)/hpl-proyecto/hpl-2.3` | Ruta donde descomprimiste HPL — **ajústala si usaste otra ubicación** |
| `MPdir` / `MPinc` / `MPlib` | `/opt/homebrew/...` | Ruta de OpenMPI instalado vía Homebrew — ajusta si tu Homebrew está en `/usr/local` (Mac Intel) |
| `LAlib` | `-framework Accelerate` | Usa Apple Accelerate (BLAS optimizado con AMX) en vez de una BLAS genérica |
| `CC` | `mpicc` | Compilador C (Apple Clang vía OpenMPI) |
| `LINKER` | `mpifort` | Necesario porque Accelerate tiene funciones en Fortran |

## 5. Compilar

```bash
make arch=MacSilicon
```

Si compila correctamente, el ejecutable queda en:

```
bin/MacSilicon/xhpl
```

## 6. Usar los `HPL.dat` ya incluidos en el repo

Cada fase de prueba documentada en `InformeHPL.md` tiene su `HPL.dat` correspondiente guardado en `~/hpl-apple-m4/datResultados/`. Para replicar una fase específica, copia el `HPL.dat` de esa fase a `bin/MacSilicon/HPL.dat`:

```bash
cp ~/hpl-apple-m4/datResultados/HPL_fase5.dat bin/MacSilicon/HPL.dat
```

Los tres parámetros principales que cambian entre pruebas son:

- **N**: tamaño de la matriz (mayor N = más memoria y más tiempo, pero potencialmente más GFLOPS).
- **NB**: tamaño de bloque, afecta el uso de caché.
- **P × Q**: grilla de procesos MPI, donde `P × Q` debe ser igual al número de procesos (`np`) con el que se ejecuta `mpirun`.

Si quieres probar una configuración nueva, edita `HPL.dat` manualmente manteniendo los parámetros algorítmicos fijos usados en todo el informe: PFACT=2, RFACT=1, BCAST=1, DEPTH=1, NBMIN=4, NDIV=2, SWAP=2, EQUIL=1.

## 7. Ejecutar

Desde `bin/MacSilicon/`:

```bash
cd bin/MacSilicon
mpirun -np <np> ./xhpl | tee ~/hpl-apple-m4/datResultados/<nombre_de_corrida>.log
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

## Estructura del repositorio

```
hpl-apple-m4/                 # Este repositorio
├── Make.MacSilicon           # Makefile configurado para Apple Silicon + Accelerate
├── datResultados/            # HPL.dat y logs .log de cada fase de prueba
├── InformeHPL.md             # Informe completo con análisis y resultados
└── README.md                 # Este archivo

~/hpl-proyecto/hpl-2.3/       # Código fuente de HPL 2.3 (descargado de Netlib, no versionado)
├── Make.MacSilicon           # Copiado desde el repo
└── bin/MacSilicon/xhpl       # Ejecutable generado al compilar
```

## Resultado de referencia

Con la configuración óptima (N=37500, NB=256, P=1, Q=8, np=8) se obtuvo **338.53 GFLOPS**, equivalente a ~51% del rendimiento teórico máximo del chip (663.93 GFLOPS). Ver `InformeHPL.md` para el análisis completo.
