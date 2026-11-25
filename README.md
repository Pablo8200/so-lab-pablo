Aquí tienes el contenido del `README.md` actualizado con los nombres exactos de tus archivos dentro de `evidencias/p1/` (`entorno-hardware.png`, `descompresion-kit.png`, `compose-ps.png`, `variables.png`, `compose.png`) y `evidencias/p2/` (`namespaces.png`, `Cgroups-limites.png`, `calculos.png`), respetando mayúsculas/minúsculas tal como aparecen en Dolphin.

```markdown
# Laboratorio SO — Kernel Linux detrás de los contenedores

Universidad Tecnológica de Pereira · Tecnología en Desarrollo de Software

## 1. Datos

- **Nombre:** Juan Pablo Vélez García
- **Código:** [ESCRIBE AQUÍ TU CÓDIGO ESTUDIANTIL]
- **Usuario de GitHub:** Pablo8200

## 2. Mi entorno (Tabla 1)

| Dato | Hardware físico | Lo que ve Linux / WSL2 |
| :--- | :--- | :--- |
| **Núcleos** | 12 núcleos lógicos (Intel Core i5-13420H) | 12 (`nproc`) |
| **RAM** | 16 GB (15.24 GiB utilizables) | 15 GiB (`free -h`) / 15608 MiB (`free -m`) |
| **Swap** | 15 GiB | 15 GiB en el equipo / 0 en los límites de memoria de los contenedores |
| **Sistema y kernel** | ASUS Vivobook 16 | CachyOS · Kernel `7.3.4-1-cachyos` |
| **Sistema de archivos** | Disco NVMe | `btrfs` en `/` |
| **Versión de cgroups** | Equipo directo (sin WSL2) | `cgroup2fs` |
| **Límite de archivos (`ulimit -n`)** | — | 1024 en la terminal / 256 en `order-api` |

## 3. Cómo instalar y ejecutar (manual de usuario)

1. **Requisitos:** Tener Linux con Docker y `cgroups v2` listos, además de Python 3 y `curl`.
2. **Preparar y levantar el laboratorio:**
   ```bash
   cd lab
   docker compose up -d --build
   docker compose ps
   curl localhost:8000/

```

3. **Cargar variables del proceso y la red (en `bash`):**
```bash
PID=$(docker inspect -f '{{.State.Pid}}' labso-order-api-1)
CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)
BR=br-$(docker network inspect -f '{{.Id}}' labso_default | cut -c1-12)
echo "PID=$PID"; echo "CG=$CG"; echo "BR=$BR"

```


4. **Ejecutar los cálculos con medición en vivo:**
```bash
python3 calculos.py --medir | tee ../fase1/salida-calculos.txt

```



## 4. Evidencias por práctica (manual técnico)

### Práctica 1 — Conocer su máquina y levantar el entorno

**Qué observé:** Comprobé los recursos de mi equipo (`12` núcleos, `15608 MiB` de RAM y `cgroup2fs`), levanté los tres servicios (`gateway`, `order-api` y `redis`) y vi que `curl localhost:8000/` responde y suma visitas (`visita=4`, `visita=5`) mostrando `pid_ns=1`.

### Práctica 2 — Un proceso, dos identidades (namespaces y cgroups)

**Qué observé:** El mismo proceso de Python aparece con dos números distintos (`8404` afuera y `1` adentro en `NSpid`), sus espacios aislados tienen identificadores distintos a los de mi terminal y sus límites de CPU (`25000 100000`), memoria (`75497472` y `0` de swap) y procesos (`300`) quedaron aplicados. Con `calculos.py --medir` obtuve `23.9 MiB` de memoria base y `5` descriptores base, guardados en `fase1/salida-calculos.txt`.

## 5. Diagrama de red

*(Puente de red observado en la prueba: `br-c714c4c37537`)*

## 6. Resultados

* [Salida de calculos.py](https://www.google.com/search?q=fase1/salida-calculos.txt)
* [Cuestionario de la Fase 1](https://www.google.com/search?q=fase1/cuestionario.md)

## 7. Errores y tropiezos que tuve hasta ahora

* **Nombre del archivo descargado y nombre de la carpeta:** Al principio intenté usar otro nombre y otra ruta de carpeta para el `.zip`, pero el nombre del archivo al descargarse de Drive cambió y tuve un error con el nombre de la carpeta hasta que usé el archivo exacto que bajó.
* **Comandos y variables en la terminal:** Al probar comandos de comodines y variables en la terminal tuve fallos de coincidencia y de sintaxis según la shell, y lo resolví corriendo los comandos de variables en `bash`.
* **Subida a GitHub:** Al intentar hacer `push` me salió error de autenticación con contraseña y aviso de que `origin` ya existía; lo solucioné usando un token personal de GitHub con permiso de lectura y escritura en el contenido del repositorio.

