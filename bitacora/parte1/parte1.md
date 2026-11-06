
# **Bitácora — Parte 1: Kernel y arranque mínimo**
**Presentado por:**  
Mariluz Calderon Guevara - 240220231078


**Docente:**  
Rafael Pérez

**Grupo:** A  
**Semestre:** VIII  
**Asignatura:** Sistemas Operativos - Noche  
**Programa:** Ingeniería de Software  
**Fecha:** Noviembre de 2026
## **Datos del entorno**

* **Sufijo del kernel (LOCALVERSION)**: \-equipo240220231078  
* **Versión del kernel (longterm 6.18.X):** 6.18.60  
* **Sistema operativo de la VM:** Debian GNU/Linux 13 (trixie)  
* **CPU de la VM:** 2 CPU   
* **RAM de la VM:** 3,8 GiB de RAM

En esta práctica se construyó desde cero la base de un sistema Linux. Se descargó el kernel longterm 6.18.60 y se compiló con la configuración por defecto (defconfig), firmado con el sufijo \-equipo240220231078 (32 min de compilación). 

Luego se exploró su configuración con menuconfig y se compiló una versión mínima (tinyconfig, 5 min, 985K). Se escribió un programa propio, init.c, compilado de forma estática y empaquetado en un initramfs que el kernel ejecutó como PID 1 y se observó cómo recibe argumentos y variables de entorno desde la línea de comandos del kernel. También se provocó un segundo panic al terminar PID 1\. Por último, se compararon los dos kernels en tamaño (14 MB frente a 985K), opciones, tiempos y memoria disponible. 

## **Configuración y requisitos previos**

**Estructura de carpetas:**

**![][image1]**

**Tiempo de compilación del kernel defconfig:** 

![image.png][image2]

![image.png][image3]

**Recursos de la VM:** 2 CPU · 3,8 GiB de RAM

**Salida de file sobre el bzImage (E1):**

![image.png][image4]

## **Actividad 1\. Explorar la configuración del kernel (E2)**

![image.png][image5]

**P1. Local version.** El valor es \-equipo240220231078. Es el sufijo LOCALVERSION que se definió, ya se había visto en la salida de file, dentro de la versión 6.18.60-equipo240220231078.

![image.png][image6]

**P2. Número máximo de CPU.** El valor de Maximum number of CPUs (NR\_CPUS) es **64\.** El kernel está compilado para manejar hasta 64 procesadores; la VM usa solo 2\.

![image.png][image7]

**P3. ext4.** ext4 está compilado dentro del kernel (\<\*\>, no \<M\>). Si fuera módulo y el disco raíz usara ext4, el kernel no podría montar la raíz al arrancar: el módulo de ext4 estaría guardado en ese mismo disco, que aún no puede leer. Habría que cargarlo antes desde un initramfs.

![image.png][image8]

**P4. BLK\_DEV\_INITRD y SERIAL\_8250\_CONSOLE.**

* **BLK\_DEV\_INITRD** está en General setup → Initial RAM filesystem and RAM disk (initramfs/initrd) support y vale \=y. El proyecto lo necesita para cargar el initramfs con \-initrd.

![image.png][image9]

![image.png][image10]

* **SERIAL\_8250\_CONSOLE** está en Device Drivers → Character devices → Enable TTY → Serial drivers → 8250/16550 and compatible serial support → Console on 8250/16550 and compatible serial port y vale \=y. Permite usar console=ttyS0 para ver los mensajes del kernel en la terminal de QEMU.

![image.png][image11]![image.png][image12]

**P5. Opciones dentro del kernel y como módulo.** Hay **1633** opciones con \=y (dentro del kernel) y **12** con \=m (módulos). Casi todo quedó compilado dentro del kernel; solo una parte mínima es modular.

![image.png][image13]

## **Actividad 2\. Compilación de tinyconfig (E2)**

**Verificación de las 5 opciones obligatorias en build-tiny/.config** (64BIT, PRINTK, SERIAL\_8250\_CONSOLE, BLK\_DEV\_INITRD, BINFMT\_ELF, todas en \=y):

![Verificación grep tinyconfig][image14]

**Tiempo de compilación del kernel tinyconfig** (real 5m10,772s; este dato va en la tabla de la Actividad 5):

![Tiempo real compilación tinyconfig][image15]

## **Actividad 3\. Kernel sin espacio de usuario (E3)**

**P6. Líneas del arranque.**

![Líneas Linux version y Command line][image16]

![Línea Memory][image17]

* Linux version 6.18.60-equipo240220231078 (mariluz@debian) (gcc ...) \#1 ...: informa la versión exacta del kernel con el sufijo del equipo, quién lo compiló, con qué compilador (gcc 14.2.0) y cuándo.  
* Command line: console=ttyS0: son los parámetros que QEMU entregó al kernel con \-append; aquí solo le indican usar el puerto serie como consola.  
* Memory: 211012K/261624K available (...): de unos 256 MB de RAM, 211012K quedan disponibles para el sistema. El resto lo ocupan el código del kernel (18907K), sus datos (rwdata, rodata, bss, init) y la memoria reservada (48400K).

**P7. Por qué se detiene.**

![Kernel panic sin initramfs][image18] 

El kernel arrancó y reconoció el hardware, pero al final intenta montar un sistema de archivos raíz para ejecutar allí el primer programa de usuario, /init. Como QEMU no le dio ningún disco ni initramfs, ni un parámetro root=, no encuentra dónde buscarlo. Termina con VFS: Unable to mount root fs on unknown-block(0,0) y entra en kernel panic, porque sin proceso PID 1 no puede continuar.

**P8. ¿Un kernel solo es un sistema operativo usable?** No. El kernel detectó CPU, memoria y dispositivos, pero a los 8 segundos del arranque se detuvo sin dejar ningún programa en ejecución: no hay consola interactiva, ni comandos, ni forma de usar el sistema. El kernel administra el hardware, pero sin un programa en espacio de usuario no hace nada útil.

## **Actividad 4\. Primer proceso del sistema (E4)**

**4.1 Programa init.c** (con \#define EQUIPO "Equipo 240220231078"):

![Contenido de init.c][image19] 

**4.2 Compilación estática** (file indica statically linked; el binario pesa 742K):

![Compilación estática de init][image20] 

**4.3 Armado del initramfs** (con dev/console creado y empaquetado con cpio \+ gzip; resultado de 318K):

![Armado del initramfs][image21] 

**4.4 Arranque con el initramfs** (PID: 1   PPID: 0, argv\[0\] \= /init, HOME=/ y TERM=linux):

![Arranque con initramfs][image22] 

**4.5 Arranque con parámetros** (console=ttyS0 quiet hola EQUIPO=03; hola llegó como argv\[1\] y EQUIPO=03 como variable de entorno):

![Arranque con parámetros][image23] 

**P9. PID y PPID.** El programa tiene PID: 1 y PPID: 0\. El PPID es 0 porque ningún proceso lo creó: lo lanza directamente el propio kernel al terminar de arrancar. Su "padre" es el kernel (el proceso 0, que no es un proceso de usuario normal), así que getppid() devuelve 0\.

**P10. Por qué \-static.** El initramfs solo contiene /init y /dev/console: no tiene la biblioteca de C ni el cargador dinámico. Con \-static, el programa lleva dentro el código de la biblioteca que usa y no necesita encontrar nada más. Sin esa opción, el kernel no podría ejecutar /init porque no encontraría el cargador dinámico (se esperaría Failed to execute y un nuevo panic).

**P11. Parámetros recibidos.** En el arranque de 4.5, hola llegó como argumento (argv\[1\]) y EQUIPO=03 como variable de entorno. console=ttyS0 y quiet no llegaron: el kernel los reconoce y los consume él mismo. La regla que usa el es kernel: los parámetros que no reconoce se entregan a PID 1; los que tienen \= van al entorno y los que no, como argumentos.

Al arrancar sin quiet, el kernel informa que no reconoció hola ni EQUIPO=03 y que los pasará al espacio de usuario (log arranque-parametros-sin-quiet.log):

![Unknown kernel command line parameters][image24] 

**P12. Mensajes con y sin quiet.** El log del arranque sin quiet (4.4) tiene **353 líneas** y el de 4.5, con quiet, solo **21**. El parámetro quiet lo consume el propio kernel: reduce el nivel de mensajes que se muestran en consola y oculta los informativos del arranque.

![Conteo de líneas de los logs][image25] 

## **Actividad 5\. Matar a PID 1 y comparar kernels (E5)**

**5.1 Segundo kernel panic:** el init termina (parámetro salir; Attempted to kill init\! exitcode=0x00000000):

![Panic al terminar init][image26] 

### **Tabla comparativa**

| Medida | defconfig | tinyconfig |
| :---- | :---- | :---- |
| Tamaño de bzImage (MB) | 14 | 0,96 (985K) |
| Opciones dentro del kernel (=y) | 1633 | 525 |
| Tiempo de compilación (real, min) | 32,4 (32m25,730s) | 5,2 (5m10,772s) |
| Tiempo de arranque hasta PID 1 (s, mediana) | 11,569 | 10,923 |
| Memoria disponible (línea Memory:, KB) | 210692 | 246964 |

**Captura con el tamaño de los dos bzImage (ls \-lh) y el conteo de opciones \=y (grep \-c):**

![Tamaño y opciones de los dos kernels][image27] 

Tiempos de arranque (3 corridas, mediana): defconfig 24,353 / 11,569 / 8,611 s → 11,569 s; tinyconfig 10,923 / 126,472 / 3,412 s → 10,923 s. Los tiempos varían mucho entre corridas porque QEMU emula por software dentro de VirtualBox y depende de la carga de la máquina en ese instante.

**P13. Comparación de los dos *panic*.** En la Actividad 3 el kernel falló con VFS: Unable to mount root fs on unknown-block(0,0): nunca existió un PID 1, porque no había ni sistema de archivos raíz ni /init que ejecutar. En 5.1 el fallo fue Attempted to kill init\! donde PID 1 sí existió y estaba corriendo (Comm: init, PID: 1), pero terminó con return 0\. El kernel no puede continuar sin PID 1 porque es el ancestro de todos los procesos de usuario. Sin él no queda ningún espacio de usuario y el kernel se detiene en lugar de seguir sin nada que ejecutar.

**P14. Diferencias entre las configuraciones.** defconfig incluye muchos controladores, sistemas de archivos, red y soporte de módulos (1633 opciones \=y y 12 módulos), por eso su bzImage pesa 14 MB y tardó 32 min en compilar. tinyconfig solo tiene lo mínimo (525 opciones \=y): pesa 985K y compiló en 5 min. En memoria, tinyconfig deja más RAM disponible porque su código es mucho menor y reserva menos. El tiempo de arranque es similar (11,569 s y 10,923 s)

**P15. Capacidades que tinyconfig no tiene.** Para las partes siguientes se necesitará: soporte de dispositivos de bloque y controladores de disco, para usar un disco real; el sistema de archivos ext4, para formatear y montar ese disco; y soporte de red (pila TCP/IP y controlador de la tarjeta de red). tinyconfig no incluye ninguna de las tres.

## **Problemas encontrados y cómo se resolvieron**

**Arranque 4.4: Kernel panic \- not syncing: IO-APIC \+ timer doesn't work\!**

![Panic IO-APIC][image28] 

* **Causa:** QEMU emula el equipo por software dentro de VirtualBox. Al arrancar, el kernel comprueba que el reloj interrumpe a través del IO-APIC. Si la VM está ocupada o lenta en ese instante, la prueba expira y el kernel declara que el temporizador no funciona. El fallo ocurre en setup\_IO\_APIC, antes de que el kernel busque /init, y es intermitente.

* **Solución:** repetir el mismo comando sin cambios; en el siguiente intento el arranque fue correcto. Si se repitiera, el propio mensaje sugiere el parámetro noapic.

[image1]: <images/image1.png>

[image2]: <images/image2.png>

[image3]: <images/image3.png>

[image4]: <images/image4.png>

[image5]: <images/image5.png>

[image6]: <images/image6.png>

[image7]: <images/image7.png>

[image8]: <images/image8.png>

[image9]: <images/image9.png>

[image10]: <images/image10.png>

[image11]: <images/image11.png>

[image12]: <images/image12.png>

[image13]: <images/image13.png>

[image14]: <images/image14.png>

[image15]: <images/image15.png>

[image16]: <images/image16.png>

[image17]: <images/image17.png>

[image18]: <images/image18.png>

[image19]: <images/image19.png>

[image20]: <images/image20.png>

[image21]: <images/image21.png>

[image22]: <images/image22.png>

[image23]: <images/image23.png>

[image24]: <images/image24.png>

[image25]: <images/image25.png>

[image26]: <images/image26.png>

[image27]: <images/image27.png>

[image28]: <images/image28.png>