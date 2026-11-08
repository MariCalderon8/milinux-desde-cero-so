# **Bitácora - Parte 2: Espacio de Usuarios y Dispositivos**
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
* **BusyBox:** 1.37.0

En esta práctica se construyó un espacio de usuario funcional, se compila BusyBox que permite usar una consola, se arma la jerarquía de directorios de un sistema Linux, se escribe un script /init que monta /proc, /sys y /dev, y se comprueba en la práctica cómo el sistema operativo representa procesos, dispositivos e interrupciones.

## **Requisitos previos**

Verificación de que el kernel defconfig tiene DEVTMPFS, BINFMT\_SCRIPT y PROC\_FS en \=y, y estructura de carpetas del proyecto. El grep mostró las tres opciones en \=y (CONFIG\_BINFMT\_SCRIPT, CONFIG\_DEVTMPFS y CONFIG\_PROC\_FS)

![Verificación de requisitos del kernel y estructura de carpetas][image1]

## **Actividad 1\. Compilar BusyBox (E1)**

### **1.1 Descarga y configuración**

Se descargó BusyBox 1.37.0 (2,4 MB) y se generó la configuración por defecto con make defconfig.

 ![Descarga y extracción de BusyBox 1.37.0][image2]

![Configuración de BusyBox: CONFIG\_STATIC=y y CONFIG\_TC desactivado][image3]

### **1.2 Compilación e instalación en rootfs**

La compilación terminó sin errores y la instalación en rootfs creó los enlaces de cada comando. file confirma que busybox es un ejecutable ELF de 64 bits enlazado de forma estática, y ls \-lh muestra 2,3 MB. La compilación tardó 1m47,086s

![Tiempo de compilación de BusyBox: real 1m47,086s][image4]

![Inicio de la compilación de BusyBox][image5]

![Instalación en rootfs, file y ls -lh de busybox][image6]

### **1.3 Observación del resultado**

rootfs contiene bin, sbin, usr y linuxrc. En bin, cada comando es un enlace simbólico de 7 bytes que apunta a busybox, que es el único archivo real (2.332.032 bytes). La instalación creó 402 enlaces simbólicos en total.

![Contenido de rootfs, enlaces de rootfs/bin y conteo de enlaces][image7]

Verificación de los enlaces de ls, cp y mount: los tres apuntan a busybox.

![rootfs/bin/ls, cp y mount apuntan a busybox][image8]

**P1. ¿Cuánto pesa el archivo busybox y cuántos enlaces simbólicos creó la instalación? Compare con el tamaño de su init de la Parte 1\.** 

busybox pesa 2,3 MB y la instalación creó 402 enlaces simbólicos. Es unas tres veces el tamaño del init de la Parte 1 (742K), pero ese único archivo reemplaza a cientos de programas: cada uno por separado, con enlace estático, repetiría el código de la biblioteca de C.

**P2. ¿A dónde apunta rootfs/bin/ls? Explique cómo un solo programa puede comportarse como ls, cp y mount.** 

rootfs/bin/ls es un enlace simbólico que apunta a busybox. BusyBox es un binario multicall: al ejecutarse mira el nombre con el que lo llamaron y corre el comando que corresponde. Cuando se escribe ls, el enlace hace que se ejecute busybox con el nombre ls, y lo mismo pasa con cp o mount. Un solo archivo contiene el código de cientos de comandos pequeños.

## **Actividad 2\. Jerarquía de directorios y /init (E2)**

### **2.1 Carpetas estándar y consola**

Se crearon las carpetas dev, proc, sys, etc, tmp y root dentro de rootfs, y el archivo de consola con mknod.

![Carpetas estándar y dev/console creados en rootfs][image9]

### **2.2 Script de arranque /init**

Se creó rootfs/init con un script que monta /proc, /sys y /dev, pone el nombre milinux-240220231078 y abre la terminal con exec.

![Creación del script rootfs/init][image10]

cat init muestra el script completo y ls \-l init confirma el permiso de ejecución (-rwxrwxr-x, 355 bytes).

![Contenido y permisos de rootfs/init][image11]

### **2.3 Permisos y empaquetado de rootfs.cpio.gz**

![Permiso de ejecución y empaquetado de rootfs.cpio.gz][image12]

**P3. Explique para qué sirve cada carpeta creada en 2.1. ¿Cuáles de ellas estarán vacías en el archivo rootfs.cpio.gz y se llenarán solo al arrancar?** 

/dev guarda los dispositivos, /proc los datos de los procesos, /sys los del hardware, /etc la configuración, /tmp archivos temporales y /root la carpeta del administrador. /proc y /sys van vacías en el cpio y se llenan al arrancar; /dev solo trae console.

**P4. Por qué el script usa exec y PID esperado de la terminal. Prediga qué PID tendrá la terminal.** 

exec reemplaza el script por la terminal en vez de crear un proceso nuevo, así que la terminal hereda el PID 1\. Consideró que la terminal tendrá PID 1 al arrancar ya que será el primer programa.

## **Actividad 3\. Arranque y PID 1 (E3)**

El sistema arrancó con el encabezado del equipo (kernel 6.18.60-equipo240220231078, 52 procesos al iniciar) y el indicador \~ \#. echo \$\$ muestra que el identificador del proceso de la terminal es 1\.

![image.png][image13]

La lista de ps de los procesos

![image.png][image14]

mount, hostname y uname \-a muestran los tres montajes del script, el nombre y el kernel con el sufijo del equipo.

![image.png][image15]

Al escribir exit, el kernel entró en pánico porque la terminal era el PID 1\.

![image.png][image16]

**P5. ¿Qué PID muestra echo \$\$? ¿Coincide con su predicción de P4? ¿Qué procesos aparecen en ps y cuáles tienen el nombre entre corchetes?** 

echo \$\$ muestra 1\. En ps, sh es el PID 1 y no lleva corchetes, lo que concuerda con la predicción de P4. Los procesos que sí  llevan corchetes, como \[kthreadd\] (PID 2\) y \[kworker/...\], son hilos del kernel.

**P6. ¿Qué mensaje apareció al escribir exit? Relácionelo con la Actividad 5 de la Parte 1\.** 

Apareció Kernel panic \- not syncing: Attempted to kill init\!. Es lo mismo que en la Actividad 5 de la Parte 1: si el PID 1 termina, el kernel no tiene qué ejecutar y se detiene, como en este caso la terminal es el PID 1 el sistema se detiene.

**P7. Copie la salida de mount e identifique los tres sistemas de archivos que montó su script.** 

rootfs on / type rootfs (rw,size=106000k,nr\_inodes=26500)  
proc on /proc type proc (rw,relatime)  
sysfs on /sys type sysfs (rw,relatime)  
devtmpfs on /dev type devtmpfs (rw,relatime,size=106000k,nr\_inodes=2650)

El script montó proc en /proc, sysfs en /sys y devtmpfs en /dev. rootfs en / lo creó el kernel antes de ejecutar init.

## **Actividad 4\. Explorar /proc y /sys (E4)**

### **4.1 ps depende de /proc**

Con /proc montado, ps lista los procesos. Después de umount /proc, ps solo muestra el encabezado y ls /proc no muestra nada. Al volver a montarlo con mount \-t proc proc /proc, ps funciona de nuevo.

![ps con /proc montado][image17]

![ps y ls /proc después de umount /proc][image18]

![ps después de volver a montar /proc][image19]

### **4.2 Lectura del estado del sistema**

/proc/1/status muestra Name: sh y State: S (sleeping); /proc/cmdline repite las opciones de arranque (console=ttyS0 quiet). meminfo indica MemTotal de 218228 kB, cpuinfo cuenta 1 procesador y uptime marcó 146,85 segundos. /proc/filesystems lista los tipos de sistemas de archivos que conoce el kernel (sysfs, proc, devtmpfs, ext4, entre otros) y ls \-l /proc/meminfo da tamaño 0\.

![status, cmdline, meminfo, cpuinfo y uptime][image20]

![filesystems y tamaño de /proc/meminfo][image21]

Como head \-12 no llega hasta VmRSS, se consultó aparte con grep VmRSS /proc/1/status.

![VmRSS del PID 1][image22]

### **4.3 Modelo de dispositivos**

/sys/class muestra las clases de dispositivos que detectó el kernel. Hay tres interfaces de red (eth0, lo y sit0), los dispositivos de bloque loop0 a loop7 y sr0, y eth0 tiene la dirección 52:54:00:12:34:56.

![Exploración de /sys][image23]

**P8. ¿Qué mostró ps después de umount /proc?**

ps solo mostró el encabezado, sin ningún proceso. ps obtiene los procesos de las carpetas numeradas de /proc, sin /proc no tiene de dónde leer.

**P9. En /proc/1/status, identifique Name, State, PPid y VmRSS. ¿Qué programa es el PID 1 ahora y cuánta memoria física usa?**  

Name es sh, State es S (sleeping) (espera que escribas algo), PPid es 0 (no tiene padre porque lo inicia el kernel) y VmRSS es 1740 kB. El PID 1 ahora es sh, el shell de BusyBox, y usa unos 1,7 MB de memoria física.

**P10. Por qué meminfo pesa 0 y cat muestra contenido.**

El archivo no está guardado en ningún disco, el kernel arma su contenido en el momento en que se lee. Por eso su tamaño es 0 aunque cat muestre datos.

**P11. Compare MemTotal de /proc/meminfo con los 256 MB que le dio a QEMU. ¿Por qué no son iguales?** 

MemTotal es 218228 kB (unos 213 MB), menos que los 256 MB de QEMU. El kernel se reserva una parte de la memoria para sí mismo (su código y sus estructuras internas) y esa parte no entra en MemTotal.

**P12. ¿Qué interfaces de red y qué dispositivos de bloque aparecen en /sys? ¿Quién los creó, si usted no instaló nada?** 

**Interfaces de red:** eth0, lo y sit0.   
**Dispositivos de bloque:** loop0 a loop7 y sr0.   
Los creó el kernel, sus controladores detectaron el hardware que simula QEMU y lo registraron en /sys.

## **Actividad 5\. mknod frente a devtmpfs (E4)**

### **5.1 Dispositivos creados por el kernel**

Con devtmpfs montado, /dev tenía 111 archivos. En ls \-l, la primera letra indica el tipo (c carácter, b bloque) y los dos números son el mayor y el menor; por ejemplo, console es 5, 1 y loop0 es 7, 0\. /proc/devices asocia cada número mayor con su tipo de dispositivo.

![Listado de /dev y conteo de archivos][image24]

![Dispositivos de carácter en /proc/devices][image25]

![Dispositivos de bloque en /proc/devices][image26]

### **5.2 Sin devtmpfs y con mknod**

Después de umount \-l /dev solo quedó console. Con mknod se crearon /dev/null (1, 3\) y /dev/mi\_consola (5, 1). El texto enviado a mi\_consola apareció en pantalla, y lo enviado a null desapareció.

![Desmontar /dev y crear dispositivos con mknod][image27]

### **5.3 Remontar devtmpfs**

Al volver a montar devtmpfs, /dev recuperó sus 111 archivos.

![Remontar devtmpfs y contar archivos][image28]

**P13. ¿Cuántos archivos había en /dev con devtmpfs y cuántos quedaron al desmontarlo? ¿De dónde salió el que quedó?**

Había 111 archivos con devtmpfs y quedó solo 1 al desmontarlo. Ese era console, que salió del mknod que se instaló en la Actividad 2 y viene dentro de rootfs.cpio.gz.

**P14. /dev/mi\_consola no se llama console, pero el texto apareció en la pantalla. Explique qué identifica realmente a un dispositivo.** 

Lo que identifica a un dispositivo son sus números mayor y menor, no su nombre. mi\_consola tiene los mismos números que console (5, 1), así que el kernel lo mandó al mismo lugar: la pantalla.

**P15. Compare crear dispositivos con mknod y usar devtmpfs. ¿Qué problema tendría un sistema con cientos de dispositivos si los creara a mano?** 

Con mknod hay que crear cada archivo a mano y saber sus números; con devtmpfs el kernel los crea solo al detectar el hardware. Con cientos de dispositivos, hacerlo a mano sería lento, propenso a errores y no seguiría los cambios del hardware.

## **Actividad 6\. Entrada/salida e interrupciones (E5)**

### **6.1 Interrupciones del puerto serie (antes y después)**

El contador de ttyS0 está en la línea de IRQ 4 (IO-APIC, edge) y marcó 2573 en la primera lectura.

![Primera lectura de /proc/interrupts para ttyS0][image29]

En una segunda medición se leyó el contador antes y después de escribir en el teclado: pasó de **77** a **335**.

**Nota:** No se hizo la medición durante la Actividad 6 por lo que se repitió nuevamente de forma independiente sin repetir toda la prueba.

![Contador de ttyS0 antes y después de presionar teclas][image30]

### **6.2 Medición con dd**

![Salida de dd, ls y meminfo][image31]

### **Cierre**

poweroff \-f apagó el sistema (reboot: Power down) y QEMU terminó solo, de vuelta en el indicador de Debian.

![Apagado con poweroff -f][image32]

**P16. ¿Cuánto aumentó el contador de ttyS0 al presionar teclas? ¿En qué línea IRQ está? Explique qué hace el kernel en cada interrupción.** 

Aumentó 258, de 77 a 335, y está en la IRQ 4\. En cada interrupción el kernel pausa lo que hacía, atiende al puerto (lee la tecla o envía texto) y sigue donde iba. Subió más que las teclas porque el puerto también avisa cuando envía texto a la pantalla, es decir cuando se presiona enter.

**P17. ¿Qué velocidad reportó el primer dd? ¿Por qué es tan alta, si no hay disco involucrado?** 

Reportó 1,3 GB/s. Es tan alta porque /dev/zero produce ceros y /dev/null los descarta, todo dentro de la memoria y sin disco.

**P18. Dónde se guardaron los 20 MB de /tmp.**

Se guardaron en la memoria RAM, porque /tmp vive allí. MemFree bajó de unos 204100 kB (lectura de la Actividad 4\) a 183148 kB, casi los 20 MB del archivo.

## **Trabajo autónomo**

### **Árbol de directorios del rootfs**

```text
rootfs/
├── root/
├── proc/
├── usr/
│   ├── bin/
│   └── sbin/
├── tmp/
├── bin/
├── sbin/
├── dev/
├── sys/
└── etc/
```
![image.png][image33] 

### **Tabla de los tres montajes de /init**

| Montaje | Sistema de archivos | Qué muestra | Comando que dependió de él |
| :---- | :---- | :---- | :---- |
| /proc | proc | Los procesos y el estado del sistema | ps dejó de funcionar al desmontarlo |
| /sys | sysfs | Los dispositivos y el hardware | cat /sys/class/net/eth0/address mostró la dirección de la red |
| /dev | devtmpfs | Un archivo por cada dispositivo | ls /dev mostró 111 archivos, sin él quedó solo console |

### **Script /init comentado**

Se agregaron comentarios al archivo rootfs/init

### ![image.png][image34] 

### **Problemas encontrados y cómo se resolvieron**

N/A

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

[image29]: <images/image29.png>

[image30]: <images/image30.png>

[image31]: <images/image31.png>

[image32]: <images/image32.png>

[image33]: <images/image33.png>

[image34]: <images/image34.png>