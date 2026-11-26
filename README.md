
```markdown
# Laboratorio SO — Kernel Linux detrás de los contenedores

Universidad Tecnológica de Pereira · Tecnología en Desarrollo de Software

## 1. Datos

- **Nombre:** Juan Pablo Vélez García
- **Usuario de GitHub:** Pablo8200

## 2. Mi entorno (Tabla 1)  

| Dato | Hardware físico | Lo que ve Linux / WSL2 |
| :--- | :--- | :--- |
| **Núcleos** | 12 núcleos lógicos (Intel Core i5-13420H) | 12 (`nproc`) |
| **RAM** | 16 GB (15.24 GiB utilizables) | 15 GiB (`free -h`) / 15608 MiB (`free -m`) |
| **Swap / zram** | 15 GiB | `/dev/zram0` 15,2 G en el equipo; `memory.swap.max=0` en el contenedor |
| **Sistema y kernel** | ASUS Vivobook 16 | CachyOS (Linux nativo) · Kernel `7.3.4-1-cachyos` |
| **Sistema de archivos** | Disco NVMe | `btrfs` en `/` · Storage driver `overlayfs` |
| **Versión de cgroups** | Equipo directo (sin WSL2) | `cgroup2fs` (cgroups v2, driver `systemd`) |
| **Memoria y THP en el host** | — | `zswap` desactivado (`N`) · `transparent_hugepage` en `[always] madvise never` |
| **Límite de archivos (`ulimit -n`)** | — | 1024 en la terminal / 256 en `order-api` |
| **Servicios del lab** | — | `python:3.11-alpine` (libc musl) en `order-api` · `nginx/1.30.8` en puerto `8000` · Redis |

Límites del contenedor `order-api` verificados en `/sys/fs/cgroup`:

- **`memory.max`:** `75497472` (`72 MiB` = `72 × 1024 × 1024` bytes)
- **`memory.swap.max`:** `0` (sin swap para el contenedor)
- **Cuota CPU (`cpu.max`):** `25000 100000` (`0.25` núcleos por ciclo de `100 ms`)
- **Límite de procesos (`pids.max`):** `300`
- **Descriptores (`nofile`):** `256`

## 3. Cómo instalar y ejecutar (manual de usuario)

1. **Requisitos:** Linux con Docker Engine, Docker Compose y `cgroups v2` activos, además de `curl`, `strace`, `iproute2` (`ss`) y `util-linux` (`nsenter`). En CachyOS/Arch:
   ```bash
   sudo pacman -S strace iproute2 util-linux

```

2. **Levantar y verificar el laboratorio:**
```bash
docker compose up -d --build
docker compose ps
curl localhost:8000/

```


3. **Cargar variables en cada terminal nueva o tras recrear `order-api` (en `bash`):**
```bash
PID=$(docker inspect -f '{{.State.Pid}}' labso-order-api-1)
CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)
BR=br-$(docker network inspect -f '{{.Id}}' labso_default | cut -c1-12)
echo "PID=$PID"; echo "CG=$CG"; echo "BR=$BR"

```


4. **Crear carpeta de salidas antes de usar `tee` y ejecutar mediciones:**
```bash
mkdir -p salidas fase1
python3 calculos.py --medir | tee fase1/salida-calculos.txt

```


5. **Ajuste en `app.py` para la Práctica 4 en este kernel:** en la rama `/ram`, se cambió la reserva de ceros por datos no nulos (`b"\x01"`) para que el kernel no descarte las páginas antes de llegar al límite de memoria:
```python
memory_leak.append(bytearray(b"\x01") * (LEAK_MB * 1024 * 1024))

```


Tras cambiar `app.py`, se reconstruye y recrea con:
```bash
docker compose up -d --build --force-recreate order-api

```



## 4. Evidencias por práctica (manual técnico)

### Práctica 1 — Conocer su máquina y levantar el entorno

**Qué observé:** Comprobé los recursos del equipo (`12` núcleos, `15608 MiB` de RAM y `cgroup2fs`), levanté los servicios del laboratorio y confirmé que `curl localhost:8000/` responde y suma visitas (`visita=4`, `visita=5`) mostrando `pid_ns=1`.

### Práctica 2 — Un proceso, dos identidades (namespaces y cgroups)

**Qué observé:** El mismo proceso de Python aparece con dos PID (`8404` en el host y `1` dentro del namespace en `NSpid`), con identificadores de namespace distintos a los de la terminal y límites de CPU (`25000 100000`), memoria (`75497472`, `0` de swap) y procesos (`300`) aplicados. Con `calculos.py --medir` obtuve `23.9 MiB` de memoria base y `5` descriptores base en `fase1/salida-calculos.txt`.

### Práctica 3 — CPU limitada (throttling)

**Qué observé:** Con 1 petición a `/cpu` obtuve `cpu_s=1.657`, `wall_s=6.539` y `ratio_wall/cpu=3.95`. Con 2 peticiones simultáneas el ratio subió a `7.98` y `7.96` (`~12.7s`), y `nr_throttled` aumentó de `34` a `228` (`+194` periodos frenados).

### Práctica 4 — Memoria y OOM Killer

**Qué observé y cómo lo resolví:**

* **Lo que falló al inicio:** En las primeras pruebas con `bytearray(...)` de ceros, el contenedor no moría en la 4.ª llamada (`retenido=15 MiB` a `60 MiB`, o incluso hasta `120–165 MiB` al acumular llamadas, con `rss` bajando en algunas lecturas). En una prueba aislada dentro del contenedor con 12 bloques de `15 MiB` (`180 MiB` en total) usando ceros, el proceso terminó con `exit=0` y `anon` bajó entre asignaciones aun con `memory.swap.max=0` y `zswap=N`. También ocurrió que al pegar `docker events` junto a otros comandos sin `&` o en otra pestaña, quedó esperando eventos en primer plano.
* **Causa y solución:** Con kernel `7.3.4-1-cachyos` y `transparent_hugepage` en `always`, páginas llenas de ceros pueden recuperarse/descartarse sin disparar el OOM Killer. Al probar la misma asignación con datos no nulos (`bytearray(b"\x01") * (15*2**20)`), el proceso murió con `exit=137` antes de completar el 3.er bloque. Actualicé `/ram` en `app.py` para usar `b"\x01"` y reconstruí con `docker compose up -d --build --force-recreate order-api`.
* **Resultado final:** Las tres primeras llamadas a `/ram` respondieron `200` (`retenido=15 MiB`, `30 MiB` y `45 MiB`, llegando a `rss=78736 kB` en la tercera), la 4.ª devolvió `502 Bad Gateway` (`nginx/1.30.8`) y `docker events` registró `container oom` y `container die` con `exitCode=137`.

### Práctica 5 — Descriptores de archivo y sockets

**Qué observé:** Con límite `256` (`fds=6` y `rss=32724 kB` al inicio), `N=50` abrió `104` FDs (`50` en `:8080`, `50` hacia `:6379` y `connected_clients:51`) y `N=100` abrió `204` FDs (`100` en `:8080`, `100` hacia `:6379` y `connected_clients:101`). Al terminar `N=100`, `/fd` mostró `fds=106` porque el pool de Redis conservó sus 100 conexiones inactivas. Con `N=150` alcanzó el tope `256` (`148` en `:8080`, `125` hacia `:6379`, `connected_clients:126`) y aparecieron los errores `[Errno 24] No file descriptors available` (mensaje de libc `musl` en Alpine, equivalente a `EMFILE` / `Too many open files`).

### Práctica 6 — Señales: apagado amable vs. forzado

**Qué observé:** Al detener `order-api` con `SHUTDOWN_DELAY` normal tardó `3.626s`, cerró conexiones y salió con `ExitCode=0` (`Saliendo limpiamente (exit 0)`), viéndose en `strace` la señal `SIGTERM`, `clock_nanosleep` y los `close(...) = 0`. Con `SHUTDOWN_DELAY=15`, al superar el plazo de `10s` de Docker, tardó `10.370s` y terminó con `ExitCode=137` sin alcanzar a completar el cierre limpio.

## 5. Diagrama de red

*(Puente de red observado en la prueba: `br-c714c4c37537`)* 

## 6. Resultados y comparación (Prácticas 3 a 6)

* [Salida de calculos.py](https://www.google.com/search?q=fase1/salida-calculos.txt)
* [Salida de descriptores (Práctica 5)](https://www.google.com/search?q=salidas/p5-fd.txt)

| Prueba | Predicho | Medido | Observación |
| --- | --- | --- | --- |
| **Ratio `wall/cpu` (1 petición `/cpu`)** | ~`4` | `3.95` (`cpu_s=1.657`, `wall_s=6.539`) | Cuota `25000 / 100000` limita el avance de CPU |
| **Ratio `wall/cpu` (2 peticiones `/cpu`)** | ~`8` | `7.98` y `7.96` (`~12.7s`) | Dos peticiones compiten por la misma cuota |
| **Bloqueos de CPU (`nr_throttled`)** | Sube | `+194` (`34` → `228`) | El kernel frenó la ejecución al agotar cuota |
| **Llamada `/ram` con corte OOM** | 4.ª llamada | 4.ª llamada (`502 Bad Gateway`) | Con `b"\x01"` en `/ram`; las 3 primeras fueron `200` |
| **Código de salida tras OOM** | `137` | `137` (`exitCode=137`, `container oom` + `die`) | Terminación forzosa (`128 + 9`) al superar `72 MiB` |
| **Descriptores con `N=50` y `N=100**` | `104` y `204` | `104` y `204` | `4` base + `2` sockets por petición (`/slow`) |
| **Tope de descriptores con `N=150**` | `256` | `256` (`Errno 24`) | `148` en `:8080`, `125` hacia `:6379`, `126` en Redis |
| **Cierre con `SIGTERM` (`SHUTDOWN_DELAY` normal)** | ~`3s`, `ExitCode=0` | `3.626s`, `ExitCode=0` | Cierra conexiones limpiamente antes de salir |
| **Cierre con `SHUTDOWN_DELAY=15**` | ~`10s`, `ExitCode=137` | `10.370s`, `ExitCode=137` | Supera los `10s` de espera y recibe corte forzado |

## 7. Errores y tropiezos que tuve hasta ahora

* **Nombre del archivo descargado y nombre de la carpeta:** Al principio intenté usar otro nombre y otra ruta de carpeta para el `.zip`, pero el nombre del archivo al descargarse de Drive cambió y tuve un error con el nombre de la carpeta hasta que usé el archivo exacto que bajó.
* **Comandos de comodines y shell (`bash`):** Al probar comandos con comodines y variables tuve fallos de coincidencia y de sintaxis según la shell, y lo resolví ejecutando en `bash`.
* **Subida a GitHub:** Al intentar hacer `push` me salió error de autenticación con contraseña y aviso de que `origin` ya existía; lo solucioné usando un token personal de GitHub con permiso de lectura y escritura en el contenido del repositorio.
* **Memoria de ceros sin OOM en la Práctica 4:** El `bytearray` inicial de ceros no disparó el corte por memoria en mi kernel (`7.3.4-1-cachyos` con `transparent_hugepage=always`); lo resolví usando datos no nulos (`b"\x01"`) en `/ram` y recreando el contenedor con `--build --force-recreate`.
* **Comandos en primer plano, rutas y carpetas (`salidas/`):** Ejecutar `docker events` junto a otros comandos sin separarlo o sin `&` bloqueaba los siguientes comandos, ejecutar fuera de la carpeta del lab impedía que `docker compose` encontrara el archivo, y correr `tee -a salidas/p5-fd.txt` antes de crear `salidas/` daba error de directorio inexistente; lo solucioné separando terminales/comandos, estando en la carpeta del proyecto y creando `salidas/` con `mkdir -p salidas`.

```
