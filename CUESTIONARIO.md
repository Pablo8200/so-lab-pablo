
---

# Cuestionario de la Fase 1 (Sustentación 1)

## 1. Con la analogía del edificio, explique qué es un contenedor y en qué se diferencia de una máquina virtual.

En la analogía del edificio, un contenedor es como un apartamento o local dentro de un mismo edificio que comparte la estructura central, las tuberías y la acometida eléctrica —es decir, el kernel del sistema operativo del host—, pero tiene sus propias divisiones y reglas para funcionar aislado con los recursos que le corresponden. En cambio, una máquina virtual es como construir un edificio completo con su propia instalación eléctrica y tuberías independientes sobre el mismo terreno: lleva su propio sistema operativo invitado completo encima de un hipervisor, lo que ofrece mayor aislamiento pero consume más recursos y tarda más en arrancar.

## 2. En la Práctica 2 el mismo proceso tiene dos PID. ¿Qué namespace lo causa? ¿Cuáles fueron sus dos números?

Lo causa el PID namespace (`pid`), que separa el espacio de identificadores de procesos para que el proceso principal del contenedor tenga su propia numeración interna. En mi prueba, el mismo proceso apareció con el PID **`8404`** en el namespace del host y con el PID **`1`** dentro del namespace del contenedor (lo que se ve en `NSpid`).

## 3. ¿Qué significan los dos números de su cpu.max? ¿Cuánta CPU tiene la API cada 100 ms?

En mi `cpu.max` (`25000 100000`), el primer número (`25000` microsegundos, equivalentes a `25 ms`) es la cuota máxima de tiempo de CPU que el grupo puede usar, y el segundo (`100000` microsegundos, equivalentes a `100 ms`) es el período o ventana de tiempo en la que se mide esa cuota. Por lo tanto, cada `100 ms` la API dispone de **`25 ms` de CPU**, lo que equivale a **`0.25` núcleos** de un núcleo completo.

## 4. Según su calculos.py, ¿en qué llamada a /ram muere la API? ¿Por qué el exit code será 137?

La API muere en la **4.ª llamada** a `/ram`: cada petición retiene `15 MiB`, así que tras tres llamadas lleva `45 MiB` acumulados más su memoria base dentro del límite de `72 MiB` (`75497472` bytes en `memory.max`) sin swap, y en la cuarta supera el tope y dispara el OOM Killer. En mi equipo con kernel `7.3.4-1-cachyos` y `transparent_hugepage` en `always`, reservar bloques llenos de ceros (`bytearray(n)`) no provocaba el corte porque el kernel podía descartar o reutilizar esas páginas sin matar el proceso, por lo que tuve que usar datos no nulos (`b"\x01"`) para que ocuparan memoria real y forzaran el OOM. El código de salida es **`137`** porque el kernel envía `SIGKILL` (señal `9`), y en Linux los códigos de salida por señal siguen la fórmula `128 + señal` (`128 + 9 = 137`).

## 5. ¿Por qué memswap_limit debe ser igual a mem_limit? ¿Qué pasaría si pudiera usar swap?

Igualar `memswap_limit` a `mem_limit` —o mantener `memory.swap.max=0` en cgroups v2— evita que el contenedor utilice espacio de intercambio. Si pudiera usar swap, el proceso no se detendría al superar su cuota de memoria física: paginaría hacia swap, caería el rendimiento por la latencia de intercambio y se retrasaría o evitaría el disparo del OOM Killer, haciendo menos claro y predecible el comportamiento del límite de memoria.

## 6. ¿Qué es un descriptor de archivo? ¿Por qué cada /slow ocupa 2 y cuántas caben a la vez en su API?

Un descriptor de archivo (FD) es un entero que el kernel asigna a un proceso en su tabla de archivos abiertos para identificar un recurso de entrada/salida, como un archivo, una tubería o un socket de red. Cada petición a `/slow` ocupa **2** descriptores porque mantiene abierto **1 socket** hacia el cliente (vía gateway) y **1 socket** hacia Redis. Con un límite `nofile` de **`256`** y **`4`** descriptores base, la capacidad teórica es `(256 - 4) / 2 = 126` peticiones simultáneas, y en mi prueba con `N=150` llegué al límite de `256` FDs con **`125`** sockets activos hacia Redis, produciéndose `Errno 24` (`No file descriptors available`) al agotarse los descriptores disponibles.

## 7. ¿Qué diferencia hay entre SIGTERM y SIGKILL? ¿Por qué la API necesita un manejador de SIGTERM siendo el PID 1?

`SIGTERM` (señal `15`) es una petición de cierre ordenado: el proceso puede interceptarla, ejecutar su rutina de limpieza para cerrar conexiones y salir de forma limpia (`exit 0`). `SIGKILL` (señal `9`) es una terminación inmediata e incondicional ejecutada por el kernel: el proceso no puede atraparla ni ignorarla y muere sin completar su limpieza (`exit 137`). Como la API corre como **PID 1** dentro de su PID namespace, no cuenta con los comportamientos por defecto de otros procesos ante ciertas señales si no los define explícitamente, por lo que necesita su propio manejador de `SIGTERM` para cerrar Redis y terminar limpiamente antes de que se cumpla el tiempo de espera de `10 s` de Docker y llegue `SIGKILL`.

## 8. Describa el camino de curl localhost:8000/ hasta Redis (Sección 1.4), nombrando el puerto 8000, el bridge y los veth.

La petición parte desde el host hacia `localhost:8000` (el **puerto 8000** expuesto en el host), pasa por el par de interfaz virtual **`veth`** conectado al **bridge** de Docker (`br-c714c4c37537`) hacia el gateway (`nginx`), desde allí cruza de nuevo por el **bridge** y su respectivo **`veth`** hacia `order-api`, y luego viaja por el **bridge** y su par **`veth`** hasta `redis`, devolviendo la respuesta por el trayecto inverso.

## 9. ¿Qué diferencia hay entre un namespace y un cgroup? Dé un ejemplo de cada uno.

Un **namespace** define **qué ve o en qué entorno aislado vive** un proceso, mientras que un **cgroup** controla **cuánto recurso puede consumir** un grupo de procesos.

* **Ejemplo de namespace:** el PID namespace, donde el proceso de `order-api` se ve como `PID 1` dentro del contenedor y como `8404` desde el host.
* **Ejemplo de cgroup:** el límite `memory.max = 75497472` (`72 MiB`) o `cpu.max = 25000 100000` aplicado al cgroup v2 del contenedor.

## 10. ¿Qué diferencias hay entre el hardware de su equipo y lo que ve Linux (Tabla 1)? Si usa WSL2, ¿qué controla .wslconfig?

En mi equipo (ASUS Vivobook 16, `12` núcleos lógicos Intel Core i5-13420H, `16 GB` físicos / `15.24 GiB` utilizables), Linux (`CachyOS`, kernel `7.3.4-1-cachyos`) ve **`12`** núcleos en `nproc`, **`15608 MiB`** (`15 GiB`) en `free -m` / `free -h` por reserva de hardware/firmware y conversión de unidades, **`15 GiB`** de swap/zram (`/dev/zram0` de `15.2 G` en el host frente a `0` en los límites de memoria del contenedor), **`btrfs`** en `/` con `overlayfs` en Docker y **`cgroup2fs`** (cgroups v2). En mi caso trabajo directamente en CachyOS nativo y no en WSL2; si se usara WSL2, el archivo `.wslconfig` en el directorio de usuario de Windows controla los límites globales de la VM de WSL2, como la RAM máxima, la cantidad de procesadores virtuales, el swap y parámetros de red o del kernel.

---

