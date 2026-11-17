# **Exploración de la planificación de procesos en XV6**

|  |  |
| ----- | ----- |
| **Integrante 1** | Jorge Luis Navia López |
| **Integrante 2** | Paulo Esteban Ordóñez Gutiérrez |
| **Repositorio (fork)** | https\://github.com/jorgenavia-pzk/xv6-riscv |
| **Servicio de IA y modelo** | Claude (Anthropic), claude.ai / app de Claude. Modelo: Claude Sonnet 5.5 |
| **Enlace a la conversación (Prompt 1)** | https://claude.ai/share/b6d4c78e-088c-457e-b652-8358ac5faf7d |
| **Fecha de la consulta con IA** | 16/11/2026 (diálogo del Prompt 1); conversación de apoyo desde el 13/11/2026 |

---

## **1\. Repositorio y entorno (25 %)**

### **1.1 Datos de partida**

| Dato | Valor |
| ----- | ----- |
| Sistema donde se clonó | Ubuntu 26.04.1 LTS (WSL2 sobre Windows) |
| Kernel de Linux | `Linux 6.18.33.2-microsoft-standard-WSL2 x86_64` |
| Commit de partida | `fa2ddcb7ca87a77af579380a6eafd5aff163b44f` |
| Fork | https\://github.com/jorgenavia-pzk/xv6-riscv |

### **1.2 Herramientas y versiones**

| Herramienta | Versión |
| ----- | ----- |
| git | 2.53.0 |
| GNU Make | 4.4.1 |
| `riscv64-linux-gnu-gcc` | 15.2.0 (Ubuntu 15.2.0-16ubuntu1) |
| GNU ld (binutils) | 2.46 |
| QEMU (`qemu-system-riscv64`) | 10.2.1 (Debian 1:10.2.1+ds-1ubuntu3.3) |
| gcc nativo | 15.2.0 |

### **1.3 Instrucciones reproducibles**

```sh
# 1. Instalar WSL con Ubuntu (en PowerShell)
wsl --install -d Ubuntu

# 2. Dentro de Ubuntu: instalar dependencias
sudo apt update
sudo apt install -y git build-essential gdb-multiarch qemu-system-riscv \
    gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu

# 3. Clonar el fork
git clone https://github.com/jorgenavia-pzk/xv6-riscv.git
cd xv6-riscv
git rev-parse HEAD

# 4. Compilar y ejecutar con una CPU
make qemu CPUS=1
```

Observación sobre el entorno: en Ubuntu 26.04 `qemu-system-misc` no incluye `qemu-system-riscv64`; fue necesario instalar `qemu-system-riscv`.

---

## **2\. Compilar y ejecutar XV6**

### **2.1 Evidencia de que XV6 está operativo**

**Comando:** `make qemu CPUS=1`, y dentro de XV6 `ls`, `cat README` y `Ctrl+P`. Se salió con `Ctrl+A`, `X`.

```
xv6 kernel is booting

init: starting sh
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2441
cat            2 3 36760
echo           2 4 35624
forktest       2 5 18168
grep           2 6 44216
init           2 7 36008
kill           2 8 35544
ln             2 9 35336
ls             2 10 42912
mkdir          2 11 35600
rm             2 12 35584
sh             2 13 58312
stressfs       2 14 36448
usertests      2 15 213544
grind          2 16 51856
wc             2 17 37760
zombie         2 18 34880
logstress      2 19 37592
forphan        2 20 36384
dorphan        2 21 35832
sync           2 22 34976
console        3 23 0
$ cat README
xv6 is a re-implementation of Dennis Ritchie's and Ken Thompson's Unix
Version 6 (v6).  xv6 loosely follows the structure and style of v6,
but is implemented for a modern RISC-V multiprocessor using ANSI C.

ACKNOWLEDGMENTS

xv6 is inspired by John Lions's Commentary on UNIX 6th Edition (Peer
to Peer Communications; ISBN: 1-57398-013-7; 1st edition (June 14,
2000)).  See also https://pdos.csail.mit.edu/6.1810/, which provides
pointers to on-line resources for v6.

ERROR REPORTS

Please send errors and suggestions to Frans Kaashoek and Robert Morris
(kaashoek,rtm@mit.edu).  The main purpose of xv6 is as a teaching
operating system for MIT's 6.1810, so we are more interested in
simplifications and clarifications than new features.

BUILDING AND RUNNING XV6

You will need a RISC-V "newlib" tool chain from
https://github.com/riscv/riscv-gnu-toolchain, and qemu compiled for
riscv64-softmmu.  Once they are installed, and in your shell
search path, you can run "make qemu".
$
1 sleep  init
2 sleep  sh
QEMU: Terminated
```

El mensaje final `QEMU: Terminated` aparece porque se salió del emulador con `Ctrl+A`, `X`; no es un error de XV6.

### **2.2 Una sola CPU virtual**

* **Cómo se establece el número de CPU:** en el `Makefile` (línea 174-175) la variable `CPUS` vale 3 por defecto si no se define (`ifndef CPUS` / `CPUS := 3`). En la línea 178, `QEMUOPTS` la pasa a QEMU con la opción `-smp $(CPUS)`. Al invocar `make qemu CPUS=1`, el valor se sobrescribe y QEMU recibe `-smp 1`.  
* **Comando usado:** `make qemu CPUS=1`

Salida de `grep -n "CPUS" Makefile`:

```
$ grep -n "CPUS" Makefile
174:ifndef CPUS
175:CPUS := 3
178:QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
```

Línea que imprimió `make` al ejecutar con `CPUS=1`:

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

Con `CPUS=1`, QEMU recibe `-smp 1` y la máquina virtual tiene una sola CPU, por lo que todos los procesos de XV6 compiten por el mismo procesador y el planificador decide cuál lo usa en cada momento.

---

## **3\. Investigación del planificador original (30 %)**

### **3.1 ¿Qué algoritmo o política usa XV6 por defecto?**

**Respuesta:** XV6 usa por defecto una política de planificación **Round Robin**. No es FIFO: en FIFO un proceso conservaría la CPU hasta terminar, y aquí cada proceso recibe un turno corto (quantum) y cuando se le acaba cede la CPU para que la use el siguiente, sin importar cuánto más necesite.

Esto se ve en `scheduler()` (`kernel/proc.c`). El `for` interno empieza en `proc[0]` y recorre la tabla de procesos de uno en uno, en orden ascendente de posición. En cada posición, el `if` solo elige al proceso si está en estado RUNNABLE. Al terminar la tabla, el `for (;;)` externo, que es infinito, empieza de nuevo desde `proc[0]`. Así, tres procesos RUNNABLE en las posiciones 3, 7 y 10 usan la CPU en el orden 3, 7, 10, 3, 7, 10, y así sucesivamente.

**Cómo se evidencia en el comportamiento del sistema:** Si el planificador fuera Round Robin, cada proceso RUNNABLE debería usar la CPU por turnos, en el orden de su posición en `proc[]`, y la CPU debería detenerse cuando ninguno esté RUNNABLE. Ambas cosas se leen en `scheduler()`: el `for` recorre `proc[]` en orden y el `if` solo elige a los RUNNABLE, y si `found == 0` se ejecuta `wfi`. Es una evidencia indirecta: no se observó la alternancia en el sistema en ejecución, solo el código y el estado de los procesos con `Ctrl+P`.

**Evidencia:** No instrumentamos el kernel. La evidencia es la lectura de `scheduler()`, donde el orden de elección depende de la posición en `proc[]` y no de una prioridad, y el estado de los procesos que muestra `Ctrl+P` en un XV6 con una sola CPU.

Salida de `Ctrl+P` ya obtenida:

```
1 sleep  init
2 sleep  sh
```

**Interpretación:** En `Ctrl+P`, `init` y `sh` están en SLEEPING: esperan algún evento. Como ninguno está RUNNABLE, en esa vuelta el `if` nunca se cumple, `found` queda en 0 y `scheduler()` ejecuta `wfi`: la CPU se detiene hasta que llegue una interrupción. Es coherente con un sistema sin trabajo pendiente.

### **3.2 ¿En qué archivo y función se selecciona el siguiente proceso?**

* **Archivo:** kernel/proc.c  
* **Función:** `scheduler()`, líneas 428 a 470  
* **Fragmento:**

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for (;;) {
    // The most recent process to run may have had interrupts
    // turned off; enable them to avoid a deadlock if all
    // processes are waiting. Then turn them back off
    // to avoid a possible race between an interrupt
    // and wfi.
    intr_on();
    intr_off();

    int found = 0;
    for (p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if (p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Don't re-enable interrupts on release.
        mycpu()->intena = 0;

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
        found = 1;
      }
      release(&p->lock);
    }
    if (found == 0) {
      // nothing to run; stop running on this core until an interrupt.
      asm volatile("wfi");
    }
  }
}
```

* **Explicación:** En `scheduler()`, el `for` interno recorre la tabla `proc[]` de uno en uno desde `proc[0]`. El `if (p->state == RUNNABLE)` decide si un proceso puede ser elegido. Si lo está, pasa a RUNNING, la CPU anota `c->proc = p` y `swtch` guarda el contexto del planificador y carga el del proceso, que desde ahí corre. Cuando el proceso devuelve la CPU, `swtch` regresa, `c->proc` vuelve a 0 y se sigue con el siguiente `p`. Al terminar la tabla, el `for (;;)` externo repite la vuelta sin fin.

### **3.3 Relación con los estados de los procesos y los eventos que ceden o recuperan la CPU**

* **Estados** (`enum procstate`, `kernel/proc.h`): `UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE`  
* **Proceso que sale de la CPU:** RUNNING → RUNNABLE (`yield`), RUNNING → SLEEPING (`sleep`), RUNNING → ZOMBIE (`kexit`).  
* **Proceso que entra a la CPU:** RUNNABLE → RUNNING (`scheduler`).  
* **Eventos que ceden la CPU:** interrupción de reloj, espera de algo y terminar.  
* **Eventos que la recuperan:** `wakeup`, `kfork` (proceso nuevo) y `kkill` (línea 614), que pasa a RUNNABLE a un proceso que estaba SLEEPING y fue marcado para morir, para que pueda terminar.

| Archivo | Función | Línea | Qué aporta |
| ----- | ----- | ----- | ----- |
| `kernel/proc.c` | `scheduler` | 447, 451 | Revisa si el proceso está RUNNABLE (línea 447\) y, si lo está, lo pasa a RUNNING (línea 451\) para que empiece a usar la CPU. |
| `kernel/proc.c` | `yield` | 505 | Cuando al proceso se le acaba el turno, pone su estado en RUNNABLE: cede la CPU, pero sigue listo. |
| `kernel/proc.c` | `sleep` | 568 | Pone el estado en SLEEPING: el proceso cede la CPU porque espera algo. |
| `kernel/proc.c` | `kexit` | 358 | Pone el estado en ZOMBIE: el proceso terminó y no volverá a usar la CPU. |
| `kernel/proc.c` | `wakeup` | 590 | Cuando llega lo que esperaban, pasa los procesos de SLEEPING a RUNNABLE, y así vuelven a poder ser elegidos. |
| `kernel/proc.c` | `kkill`  | 614  | Si el proceso que se va a matar estaba SLEEPING, lo pasa a RUNNABLE para que vuelva a poder ser elegido y salga.  |
| `kernel/trap.c` | llama a `yield()` | 86 y 158 | Cuando llega una interrupción de reloj, llama a `yield()` para quitarle la CPU al proceso actual. |

### **3.4 ¿Cómo se incorpora el código del planificador a la versión final compilada?**

`kernel/proc.c` se compila con el `gcc` cruzado de RISC-V (`-c`) y produce `kernel/proc.o`. La regla de la línea 94 del Makefile enlaza todos los `.o` de `OBJS`, incluido `proc.o`, con el script `kernel.ld` y produce `kernel/kernel`. `make qemu` (línea 183\) depende de `kernel/kernel`, y QEMU lo carga con `-kernel kernel/kernel`; así el código de `scheduler()` queda dentro del kernel que corre en la máquina virtual.

```
4:OBJS = \
15:  $K/proc.o \
94:$K/kernel: $(OBJS) $K/kernel.ld
95:        $(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS)
183:qemu: check-qemu-version $K/kernel fs.img
```

---

## **4\. MLFQ: descripción y exploración (30 %)**

### **4.1 Descripción inicial de MLFQ (escrita ANTES de consultar a la IA)**

**Reglas que conocemos:**

1. Si la prioridad de A es mayor que la de B, corre A y B no.  
2. Si A y B tienen la misma prioridad, corren en Round Robin, cada uno con un quantum.  
3. Cuando un trabajo entra al sistema, se pone en la cola de mayor prioridad.  
4. Si un proceso usa la CPU por mucho tiempo en un nivel alto de prioridad, después de unas cuantas veces el sistema operativo lo degrada para darle tiempo a otros procesos.  
5. Después de un periodo S, el sistema operativo puede subir de prioridad a algunos procesos.

**Dudas identificadas:**

* ¿Cuántas veces, o cuánto tiempo, puede un proceso usar la CPU en un nivel antes de bajar?  
* ¿Cada cuánto es el periodo S y a qué procesos se les sube la prioridad?

### **4.2 Prompt 1 (enviado en conversación nueva)**

```
Actúa como tutor socrático de una pareja de estudiantes de cuarto semestre de Ingeniería de Sistemas. Conocemos C++ y hemos comenzado a explorar XV6 para RISC-V.

Queremos identificar y justificar qué se necesitaría para incorporar MLFQ a XV6. En esta entrega no vamos a implementarlo.

Nuestra descripción inicial del algoritmo, escrita antes de esta conversación:

Reglas que conocemos:
1. Si la prioridad de A es mayor que la de B, corre A y B no.
2. Si A y B tienen la misma prioridad, corren en Round Robin, cada uno con un quantum.
3. Cuando un trabajo entra al sistema, se pone en la cola de mayor prioridad.
4. Si un proceso usa la CPU por mucho tiempo en un nivel alto de prioridad, después de unas cuantas veces el sistema operativo lo degrada para darle tiempo a otros procesos.
5. Después de un periodo S, el sistema operativo puede subir de prioridad a algunos procesos.

Dudas identificadas:
- ¿Cuántas veces, o cuánto tiempo, puede un proceso usar la CPU en un nivel antes de bajar?
- ¿Cada cuánto es el periodo S y a qué procesos se les sube la prioridad?
.

Lo que averiguamos del planificador original y de su incorporación al sistema compilado:

- Archivo y función donde se elige al siguiente proceso: kernel/proc.c, scheduler(), líneas 428 a 470.
- Cómo recorre los procesos y a quién elige: un for interno recorre la tabla proc[] desde la posición 0 en orden ascendente; el if elige solo a los que están RUNNABLE. Al terminar la tabla, el for(;;) externo repite desde 0. Es Round Robin.
- Estado para ser elegido: RUNNABLE. Al elegirlo pasa a RUNNING (línea 451) y se llama swtch.
- Qué hace swtch: guarda el contexto del planificador y carga el del proceso; "regresa" cuando el proceso devuelve la CPU.
- Eventos que sacan a un proceso de la CPU: interrupción de reloj (trap.c, usertrap y kerneltrap llaman a yield(): RUNNING -> RUNNABLE), espera de algo (sleep: RUNNING -> SLEEPING) y terminar (kexit: RUNNING -> ZOMBIE). Todos terminan en sched(), que vuelve al planificador.
- Eventos que lo devuelven a RUNNABLE: wakeup (SLEEPING -> RUNNABLE) y kfork (proceso nuevo en RUNNABLE).
- Cómo entra este código a la versión compilada: kernel/proc.c se compila con el gcc de RISC-V (-c) a kernel/proc.o; el Makefile (línea 94-95) enlaza todos los .o de OBJS con kernel.ld y produce kernel/kernel; make qemu (línea 183) depende de kernel/kernel y QEMU lo carga con -kernel kernel/kernel.
- Lo que todavía no tengo claro: sé que la interrupción de reloj es la que hace que el proceso suelte la CPU, pero no entiendo cómo funciona por dentro: cómo se genera y cómo llega hasta trap.c.
.

Reglas de interacción:

1. Formula una sola pregunta principal por turno y espera nuestra respuesta. No presentes una lista completa de cambios ni una arquitectura resuelta.
2. Empieza examinando nuestra descripción de MLFQ. Ayúdanos a detectar reglas ambiguas, ausentes o contradictorias. No sustituyas nuestra descripción por una versión terminada ni elijas silenciosamente una variante del algoritmo.
3. Pídenos que propongamos y justifiquemos las decisiones que faltan. Después, ayúdanos a comprobarlas con ejemplos pequeños de procesos y decisiones de planificación que nosotros debamos resolver.
4. Guíanos para relacionar cada regla con XV6. Antes de sugerir una modificación, pregúntanos qué comportamiento necesitamos y qué evidencia del código respalda nuestra propuesta.
5. No supongas que tienes acceso a nuestro repositorio. Pide los fragmentos necesarios y distingue lo observado de lo que todavía sea una hipótesis. No inventes archivos, funciones o comportamientos de nuestra versión.
6. No entregues código de implementación, parches, una lista cerrada de archivos por modificar ni pseudocódigo que resuelva el planificador. Revisa nuestras propuestas y señala problemas mediante preguntas y pistas graduales. Puedes explicar conceptos previos con ejemplos mínimos.
7. Si nos bloqueamos, reduce la dificultad de la pregunta y ofrece una pista concreta. Si pedimos la solución directa, conserva el modo socrático.

El objetivo es que nosotros logremos:

A. Redactar una descripción revisada y coherente de la variante de MLFQ considerada.
B. Explicar qué cambios necesitaría XV6 para soportarla, indicando para cada uno su propósito, su relación con las reglas y la evidencia del código que lo justifica.
C. Proponer escenarios para comprobar una futura implementación del algoritmo MLFQ en el sistema operativo XV6. Pídenos que expliquemos qué procesos participarían, qué harían, qué regla comprobaríamos y qué comportamiento esperaríamos observar. No nos entregues los escenarios resueltos. En esta tarea solo los describiremos; no vamos a programarlos ni ejecutarlos.

Mantén el análisis dentro del alcance de una propuesta: no necesitamos implementar MLFQ ni construir un selector de planificadores. Ayúdanos a registrar las cuestiones que queden pendientes de investigación.

Al finalizar, pídenos que redactemos nuestras conclusiones. Comprueba su consistencia mediante una última pregunta aplicada a un caso concreto, sin escribir por nosotros el reporte.

Comienza con una pregunta que permita comprobar cómo entendemos una regla de nuestra descripción inicial de MLFQ.

```

### **4.3 Descripción revisada y consolidada de la variante de MLFQ**

| Campo | Decisión |
| ----- | ----- |
| Número de colas | 3 niveles de prioridad: alto, medio y bajo (mínimo 3; el número exacto está pendiente, ver 4.5). |
| Política en cada nivel | Round Robin con quantum entre los procesos del mismo nivel (regla 2 de la descripción inicial). |
| Regla de asignación inicial | Un proceso nuevo entra en la cola de mayor prioridad (regla 3). |
| Regla de bajada | La degradación se mide por el tiempo acumulado en el nivel. Un proceso baja de nivel al acumular 30 ms (3 quantums) en él. Se revisa en cada tick, así que baja aunque no haya agotado su quantum actual. Al entrar a un nivel nuevo, el acumulado empieza en 0 (decisión provisional). |
| Regla de subida / boost | Cada periodo S \= 100 ms, todos los procesos suben y se reinicia su tiempo acumulado. El sistema recuerda el instante del último ascenso. |
| Quantum por nivel | 10 ms. |
| Preferencia entre niveles | Un proceso de mayor prioridad desaloja al de menor prioridad en el siguiente tick, a lo sumo 1 tick después de quedar RUNNABLE (porque el desalojo ocurre en la interrupción de reloj; ver caso D y E en 4.8). |
| Proceso que cede la CPU antes de agotar el quantum | Si se bloquea por E/S, vuelve a RUNNABLE en el mismo nivel y su tiempo acumulado se conserva y sigue sumando (decisión provisional). |
| Turno dentro de un nivel | Los procesos del mismo nivel se turnan. XV6 no tiene colas, así que hay que construir ese turno (pendiente 5 en 4.5). |

### **4.4 Propuesta de cambios en XV6 (teórica, sin implementar)**

| \# | Archivo / función | Cambio propuesto | Necesidad que atiende y por qué | Regla de MLFQ relacionada | Evidencia en el código |
| ----- | ----- | ----- | ----- | ----- | ----- |
| 1 | `kernel/proc.h`, `struct proc` | Agregar un dato por proceso con su nivel de prioridad. | Hoy el planificador no sabe en qué nivel está cada proceso, y sin eso no puede comparar prioridades. | Reglas 1 y 3 | Observado: en `struct proc` (líneas 82 a 104\) no hay ningún campo de prioridad, nivel ni tiempo. Fragmento completo debajo de la tabla.  |
| 2 | `kernel/proc.h`, `struct proc` | Agregar un dato por proceso con el tiempo acumulado en su nivel actual (empieza en 0 al entrar a un nivel nuevo). | Sin ese dato no se puede decidir cuándo un proceso baja de nivel, ni reiniciarlo cada S. | Reglas 4 y 5 | Observado: no hay campo con ese propósito en `struct proc`. |
| 3 | `kernel/proc.c`, `scheduler()` (líneas 428 a 470\) | Que la elección dependa del nivel y, dentro del nivel, de un turno que recuerde el último proceso elegido, y no solo de la posición en `proc[]`. | Hoy un RUNNABLE de la posición 5 corre antes que uno de la 12 aunque el segundo tuviera mayor prioridad. | Reglas 1 y 2 | Observado: el `for` (línea 445\) recorre `proc[]` en orden ascendente y el `if` (línea 447\) solo mira `p->state == RUNNABLE`. Hipótesis: cómo recordar el último elegido por nivel está sin resolver (pendiente 5). |
| 4 | `kernel/trap.c`, `usertrap` y `kerneltrap` (líneas 86 y 158\) | Sumar el tiempo al proceso que está corriendo en cada interrupción de reloj. | Alguien tiene que medir el tiempo acumulado, y la interrupción de reloj ocurre sola y de forma periódica mientras el proceso corre. | Regla 4 | Observado: `yield()` se llama desde `trap.c` solo cuando `which_dev == 2` (interrupción de reloj); es el único evento visto que obliga a un proceso a soltar la CPU sin que él lo pida (`sleep` y `kexit` son voluntarios). Hipótesis: ahí se podría sumar el tiempo. |
| 5 | Por definir (hipótesis: `kernel/trap.c`, en `clockintr()`, o `kernel/proc.c`) | Una rutina que cada S suba a todos los procesos y reinicie su tiempo acumulado, apoyada en un contador global de ticks y en el instante del último ascenso. | Evitar que un proceso de nivel bajo quede sin CPU indefinidamente (inanición). | Regla 5 | Observado: `uint ticks` (línea 10 de `trap.c`) y `clockintr()` (líneas 167 a 173\) ya hacen `ticks++`. Hipótesis: no se ha verificado dónde se ejecutaría el ascenso ni cómo respetar el `p->lock` de cada proceso. |
| 6 | `kernel/proc.h`, `struct proc` | Agregar un tercer dato por proceso: el tiempo usado en el quantum actual. | Sin ese dato no se pueden medir los 10 ms ni repartir el turno dentro del nivel. | Regla 2 | Observado: no hay campo con ese propósito en `struct proc`. |

Fragmento de `struct proc` (`kernel/proc.h`, líneas 82 a 104), donde se ve que no hay ningún campo de nivel, prioridad ni tiempo:

```
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state; // Process state
  void *chan;           // If non-zero, sleeping on chan
  int killed;           // If non-zero, have been killed
  int xstate;           // Exit status to be returned to parent's wait
  int pid;              // Process ID

  // wait_lock must be held when using this:
  struct proc *parent; // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

Observado: el comentario de la línea 85 dice que `p->lock` debe estar tomado para usar `state`, `chan`, `killed`, `xstate` y `pid`. Hipótesis: los campos nuevos (nivel, tiempo acumulado, tiempo del quantum) tendrían que decidirse si se protegen con `p->lock`, lo que se relaciona con el pendiente 6 de 4.5.

### **4.5 Decisiones que necesitan investigación o soporte adicional**

1. **Duración de un tick en ms.** Leer `timerinit()` en `kernel/start.c`. Sin ese dato no se pueden traducir los ms a ticks ni acotar cuánto tarda en desalojar un proceso de mayor nivel.  
2. **Número exacto de niveles** (hoy solo "al menos 3") **y valores definitivos de quantum, límite por nivel y S.** Hay que traducir 10, 30 y 100 ms a ticks una vez conocida la duración del tick.  
3. **Tiempo acumulado tras un bloqueo por E/S.** Hoy se conserva (decisión provisional, porque así un proceso con ráfagas cortas como C igual termina bajando). Falta decidir si conviene.  
4. **Tiempo acumulado al entrar a un nivel nuevo.** Hoy empieza en 0 (decisión provisional, por el conteo por nivel). Falta verificar que sea lo adecuado.  
5. **Cómo recordar el turno dentro de un nivel** (hipótesis sin resolver, porque XV6 no tiene colas). Hay que investigar en `scheduler()` de `kernel/proc.c`, que hoy recorre `proc[]` por posición, cómo guardar el último proceso elegido en cada nivel (por ejemplo, un valor por nivel) y cómo evitar que la búsqueda siempre reinicie desde `proc[0]`, lo que dejaría sin turno a los procesos de posiciones altas.  
6. **Dónde y cómo se ejecutaría el ascenso, y cómo evitar choques con los candados.** Falta decidir en qué función iría (opciones a investigar: `clockintr()` en `trap.c`, que ya hace `ticks++`, o `scheduler()` en `kernel/proc.c`) y cómo modificar el nivel y el acumulado de cada proceso sin chocar con el `p->lock` de cada uno. Hay que revisar el orden de `acquire` y `release` de `p->lock` que ya usa `scheduler()`, para que el ascenso lo respete y no provoque un bloqueo.  
7. **Comportamiento con varias CPUs (por verificar).** El Makefile usa `CPUS := 3` por defecto y cada núcleo corre su propio `scheduler()` (`struct cpu *c = mycpu()`). Hay que verificar si el turno por nivel (pendiente 5\) y el ascenso (pendiente 6\) son estado compartido entre núcleos y cómo protegerlos con candados, y si el desalojo "en el siguiente tick" se cumple en cada núcleo. Nuestras pruebas fueron con `CPUS=1`.

### **4.6 Escenarios para comprobar una futura implementación (solo descritos, no ejecutados)**

Supuesto común: los procesos nuevos entran al nivel alto, y se usan los valores de 4.3 (quantum de 10 ms, bajada a los 30 ms acumulados en el nivel, S \= 100 ms).

| \# | Procesos que participan | Qué hace cada proceso | Regla que se comprueba | Comportamiento esperado si es correcta |
| ----- | ----- | ----- | ----- | ----- |
| E1 | A y B | A es un cálculo largo que nunca pide E/S, en el nivel alto. B está RUNNABLE en el nivel bajo. Se supone el mismo límite de 3 quantums por nivel. | Reglas 1 y 4 | B no corre mientras A esté en un nivel superior al suyo. A baja a los 30 ms (a medio) y a los 60 ms (a bajo), y desde ahí A y B se turnan dentro del nivel. B corre por la regla 4 (A baja a su nivel), no por el ascenso. |
| E2 | C y F | C empieza en el nivel alto: usa la CPU 9 ms, pide E/S, se bloquea (cada espera dura menos de 10 ms, por ejemplo 5 ms) y vuelve a RUNNABLE en el mismo nivel; repite. F es un cálculo largo que empieza en el nivel medio con 0 ms acumulados y usa la CPU mientras C está bloqueado. | Regla 4 (tiempo acumulado conservado tras E/S) | C acumula 9, 18 y 27 ms y baja al nivel medio durante su cuarta ráfaga, al llegar a 30 ms (9+9+9+3). F acumula 15 ms (3 x 5\) antes de que C baje, así que no baja antes que C (15 \< 30). Mientras C está en el nivel alto corre antes que F; desde que C baja al medio, C y F se turnan, y F baja al nivel bajo cuando complete sus 30 ms (le faltan 15). Si el acumulado se reiniciara tras la E/S o solo se contaran quantums completos, C nunca bajaría ni se turnaría con F. Simplificación: se ignora el retraso de hasta 1 tick con que C desaloja a F al despertar; depende de la duración del tick (pendiente 1). |
| E3 | B, D y una serie de procesos cortos | B está RUNNABLE en el nivel bajo. D está en el nivel medio, con 20 ms acumulados cuando llega el ascenso. Llega un proceso corto nuevo cada 20 ms; cada uno usa 20 ms de CPU y termina, así que ninguno llega a 30 ms, ninguno baja de nivel y la CPU siempre está ocupada en el nivel alto. | Regla 5: el ascenso cada S \= 100 ms sube a todos y reinicia su tiempo acumulado | Entre 0 y 100 ms corren solo los procesos cortos, uno tras otro; B y D no corren por la regla 1\. A los 100 ms el ascenso sube a B y D al nivel alto con acumulado 0\. Desde ahí B, D y los procesos que sigan llegando se turnan en el nivel alto, y D corre 30 ms propios antes de bajar. Error que detecta: que el ascenso no ocurra o llegue tarde; en el orden de la CPU se vería que B no corre nunca o solo corre después de los 100 ms. D permite además comprobar el reinicio: si el acumulado no se reiniciara, D bajaría tras solo 10 ms de CPU en el nivel alto (20 \+ 10 \= 30). |

### **4.7 Conclusiones de la pareja**

Distinguimos **propuesta** (decisión de diseño nuestra), **observado** (comprobado con casos numéricos o leyendo el código) e **hipótesis** (suposición sobre XV6 aún sin verificar). MLFQ no está implementado ni probado.

**A. Descripción revisada de MLFQ (propuestas)**

* Al menos 3 niveles; un nivel mayor tiene prioridad sobre uno menor.  
* Quantum de 10 ms. Un proceso baja de nivel al acumular 30 ms (3 quantums) en ese nivel, revisado en cada tick. Al entrar a un nivel nuevo, el acumulado empieza en 0 (provisional).  
* Cada S \= 100 ms todos suben de nivel y su acumulado se reinicia; el sistema recuerda el instante del último ascenso.  
* Un proceso de nivel mayor que pasa a ejecutable desaloja al que corre en el siguiente tick, a lo sumo 1 tick después.  
* Si un proceso cede la CPU por E/S o `sleep`, conserva su tiempo acumulado (provisional).  
* Dentro de un nivel los procesos se turnan; XV6 no tiene colas, así que ese turno hay que construirlo.

Correcciones hechas con casos numéricos (observado): "el 7 nunca correría" era falso para el planificador original, que recorre 3, 7, 9; y los 30 ms de C no cuadraban si se revisaba solo al final del quantum, por eso la revisión quedó en cada tick.

**B. Cambios que necesitaría XV6**

El más importante son los datos nuevos por proceso. Hoy `struct proc` no tiene campo de prioridad, nivel ni tiempo (observado, `kernel/proc.h` líneas 82 a 104). Hacen falta: nivel, tiempo acumulado en el nivel y tiempo usado en el quantum actual. Sin ellos ninguna regla se puede aplicar.

Cambios de apoyo (hipótesis): un contador global de ticks y el instante del último ascenso; `scheduler()` elige por nivel y por turno dentro del nivel; la cuenta de tiempo y el ascenso se harían en la interrupción de reloj (`clockintr()` ya hace `ticks++`). Observado en `trap.c`: `yield()` solo se llama con `which_dev == 2` (reloj), en `usertrap` (línea 86\) y `kerneltrap` (línea 158). La interrupción de reloj es el único evento visto que obliga a un proceso a soltar la CPU sin que él lo pida; `sleep` y `kexit` son voluntarios.

**C. Escenarios (solo descritos, no ejecutados)**

* **E1 (A y B):** comprueba las reglas 1 y 4\. B no corre mientras A esté en un nivel superior; A baja a los 30 y 60 ms, y desde ahí se turnan.  
* **E2 (C y F):** comprueba la regla 4 con el acumulado conservado tras E/S. F empieza con 0 ms acumulados en el medio y cada E/S de C dura menos de 10 ms. C baja al medio en su cuarta ráfaga y desde ahí se turna con F.  
* **E3 (B, D y procesos cortos):** comprueba la regla 5\. Entre 0 y 100 ms solo corren los procesos cortos; a los 100 ms el ascenso sube a B y D con acumulado 0\. Detecta que el ascenso no ocurra o llegue tarde, y con D, que no se reinicie el acumulado.

Cada escenario aísla una regla y puede detectar un error concreto de la implementación.

**D. Pendientes de investigación**

La lista completa está en 4.5 (siete pendientes). Los principales: la duración de un tick en ms (leer `timerinit()`), el número exacto de niveles y los valores definitivos de quantum, límite y S, las dos decisiones provisionales sobre el tiempo acumulado, cómo recordar el turno dentro de un nivel, dónde ejecutar el ascenso respecto a los candados, y el comportamiento con varias CPUs.

### **4.8 Retroalimentación de la IA**

La IA valoró la organización de las conclusiones (A, B, C y pendientes) y la documentación de las correcciones hechas con casos numéricos. Señaló lo siguiente y esto fue lo que ajustamos:

1. **Etiquetas:** mezclábamos propuestas con hipótesis. Ahora las decisiones de diseño son "propuesta" e "hipótesis" queda para lo no verificado del código.  
2. **Afirmación de más:** dijimos que la interrupción de reloj era el único punto por el que se llega a un proceso en ejecución. La reformulamos con lo observado en `trap.c` (`yield()` solo con `which_dev == 2`).  
3. **Sección C:** solo nombraba E1, E2 y E3. La completamos con procesos, acciones, regla y comportamiento esperado.  
4. **Caso final (`kfork` o `wakeup` y el desalojo de D):** la regla A decía "de inmediato", pero el mecanismo de B solo actúa en el tick de reloj. Corregimos la regla A: "siguiente tick, a lo sumo 1 tick".  
5. **E1 no aislaba la regla 5:** B corre a los 60 ms por la regla 4\. E1 quedó para las reglas 1 y 4 y la regla 5 se comprueba en E3.  
6. **E2:** F "fijo" en el nivel medio no lo respaldaba la regla 4\. F empieza con 0 ms acumulados y cada E/S de C dura menos de 10 ms.  
7. **Acumulado en un nivel nuevo:** no estaba decidido. Lo fijamos en 0 (provisional).  
8. **Pendientes que faltaban en D:** turno dentro de un nivel, ascenso y candados, y varias CPUs. Los agregamos.

---

## **5\. Evidencia del aprendizaje con IA (15 %)**

**Servicio de IA y modelo:** Claude (Anthropic), modelo Claude Sonnet 5.5, en dos usos: (1) el diálogo socrático del Prompt 1, en una conversación nueva (16/11/2026); (2) una conversación de apoyo, aparte, para configurar el entorno, entender el código del planificador y organizar el reporte.

### **Intercambio 1: \[hipótesis\]**

* **Pregunta de la IA:** A está en el nivel de mayor prioridad y es un cálculo largo que nunca pide E/S ni espera nada. B está en un nivel de menor prioridad y está RUNNABLE. Aplicando solo la regla 1, ¿qué le pasa a B? Luego revisen las otras cuatro reglas y digan cuáles, si alguna, cambian ese resultado y por qué.  
* **Nuestra respuesta:** «Lo que le pasa a B y está runable yo creería que después de un tiempo subiría de prioridad, ya que el sistema en algunos momentos puede subir algunos procesos que están en prioridad baja a prioridad alta, mientras que A, al ser un cálculo largo que no hace procesos o iteraciones de entrada y salida, va degradando su prioridad.»  
* **Qué aprendimos / cómo lo comprobamos:** Aprendimos que mi hipótesis mezclaba dos cosas: A baja por la regla 4, no solo B sube por el ascenso. Lo comprobamos con números en E1: B corre a los 60 ms por la regla 4\. Por eso E1 quedó para las reglas 1 y 4 y la regla 5 se prueba en E3.

### **Intercambio 2: \[comprobación\]**

* **Pregunta de la IA:** D y E están RUNNABLE y llega la interrupción de reloj. D ocupa la posición 5 de `proc[]` y E la posición 12\. Según su descripción de cómo `scheduler()` recorre la tabla y a quién elige, ¿a cuál de los dos elegiría el XV6 original? ¿Importa que E tenga «mayor prioridad» en su diseño? Con eso, díganme qué información le falta hoy al planificador para cumplir su regla 1\.  
* **Nuestra respuesta:** «En XV6, en este caso, no sería el que tuviera mayor prioridad, sino el orden de la lista. Entonces, en este caso, los dos están runnable, entonces el primero en ser ocupado por la CPU sería el D y luego sería E.»  
* **Qué aprendimos / cómo lo comprobamos:** Aprendimos que el XV6 original elige por posición en `proc[]` (D antes que E), sin ningún dato de prioridad en `struct proc`. Lo comprobamos leyendo el `for` de `scheduler()` (`kernel/proc.c`, línea 445\) y los campos de `struct proc` (`kernel/proc.h`, líneas 82 a 104).

### **Intercambio 3: \[corrección\]**

* **Pregunta de la IA:** Elijan tres valores y justifíquenlos: la duración del quantum, cuántos quantums puede gastar un proceso en un nivel antes de bajar, y el periodo S.  
* **Nuestra respuesta:** «La duración del quantum puede ser en milisegundos. ¿Cuántos quantum puede gastar un proceso en un nivel antes de bajar? Pueden ser 10 milisegundos, dependiendo, me imagino, del algoritmo.»  
* **Qué aprendimos / cómo lo comprobamos:** Aprendimos que elegir los valores exigía justificarlos con casos numéricos, y esos casos corrigieron los 30 ms de C y el "el 7 nunca correría". Lo comprobamos haciendo las cuentas paso a paso.

### **Reflexión individual**

**Jorge Luis Navia López:** Comprendí que XV6 usa Round Robin: `scheduler()` en `kernel/proc.c` recorre `proc[]` por posición y elige solo a los procesos RUNNABLE, sin ninguna noción de prioridad. También entendí cómo un proceso entra y sale de la CPU (`yield`, `sleep`, `kexit`, `wakeup`) y cómo ese código llega al kernel compilado a través del Makefile. Con MLFQ aprendí que mi hipótesis inicial mezclaba dos cosas: A baja de nivel por la regla 4, y B no tiene que esperar al ascenso de la regla 5\. Aporté la descripción inicial de las reglas y las decisiones sobre quantum, límite por nivel y S. Resolví con casos numéricos las correcciones de los 30 ms de C y del “el 7 nunca correría”. Aún necesito estudiar cómo funciona la interrupción de reloj por dentro (`timerinit()` y cómo llega a `trap.c`), cómo recordar el turno dentro de un nivel sin colas, y cómo hacer el ascenso respetando el `p->lock` de cada proceso. 

**Paulo Esteban Ordóñez Gutiérrez:** Comprendí que lo que decide quién corre en XV6 es la posición en la tabla `proc[]` y no una prioridad, porque `struct proc` no tiene ningún campo de nivel ni de tiempo. Por eso MLFQ exige datos nuevos por proceso (nivel, tiempo acumulado y tiempo usado en el quantum), y que `scheduler()` y la interrupción de reloj trabajen con ellos. Aporté el análisis del caso de D y E (el XV6 original elige primero a D por su posición, aunque E tenga mayor prioridad en nuestro diseño). También ayudé a distinguir entre propuesta, observado e hipótesis, y a describir los escenarios E1, E2 y E3 con la regla que comprueba cada uno. Aún necesito estudiar cómo se comportaría MLFQ con varias CPUs, porque cada núcleo corre su propio `scheduler()` y el turno por nivel y el ascenso serían estado compartido. También debo estudiar cómo traducir ms a ticks, una vez que se conozca la duración real de un tick. 

---

 