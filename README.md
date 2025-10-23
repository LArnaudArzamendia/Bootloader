# BIOS
La BIOS (legacy) va al _primer sector físico_ del disco booteable y copia los primeros **512 bytes** directo a la RAM. Convencionalmente, lo hace en la dirección física **0x7C00**. Aunque, esto puede variar, y es solamente una convención.

## Pero, ¿qué hay en esos 512 bytes?
Lo que se denomina como el **Master Boot Record** (MBR), en el ***cilindro 0***, ***head 0*** y ***sector 1*** (se basa en 1, no en 0) -> (o ***LBA 0***) del disco.
El MBR contiene 3 cosas:
- El código del bootloader (desde el byte 1, hasta el byte 446).
- La tabla de particiones del disco, una estructura de datos de 64 bytes (desde el byte 447, hasta el byte 510). Contiene la descripción de un total de ***4 particiones*** (de 16 bytes, cada una).
- La ***firma mágica***, indicando que es un sector de arranque válido, definido cuando un disco es booteable. Corresponden a **0xAA55** (bytes 511 y 512). En ***Little-Endian*** se escriben como **55 AA**.

### Tabla de particiones
#### Offset de particiones
- Offset 0x01BE (byte 447).
- Offset 0x01CE (byte 463).
- Offset 0x01DE (byte 479).
- Offset 0x01EE (byte 495).

#### Elementos de cada partición
- Offset 0x00 (1 byte): Estado de arranque (bootable flag).
- Offset 0x01 (1 byte): Head inicial.
- Offset 0x02 (6 bits): Sector inicial.
- Offset 0x03 (10 bits): Cylinder inicial.
- Offset 0x04 (1 byte): System ID (indica el tipo de filesystem contenido en la partición - Ej: Ext2, FAT32, NTFS, ...).
- Offset 0x05 (1 byte): Head final.
- Offset 0x06 (6 bits): Sector final.
- Offset 0x07 (10 bits): Cylinder final.
- Offset 0x08 (4 bytes): LBA de inicio de la partición.
- Offset 0x0C (4 bytes): Total de sectores en la partición.

# Discos
Los discos son dispositivos de almacenamiento de datos. Hoy en día, tenemos ***unidades de disco duro*** (**HDD**), que usan discos magnéticos giratorios, y las ***unidades de disco sólido*** (**SSD**), que emplean memoria flash.

## Anatomía de un disco duro
- **Platter (Plato)**: Es un disco físico que almacena datos. Se encuentran apilados uno tras otro en una ruta vertical, para formar el disco duro. ***Contiene a las pistas***.
- **Head (Cabezal)**: Componente físico que lee y escribe los datos en las pistas. Se ubican al extremo del ***brazo***, que los posiciona sobre las partículas magnéticas. **Hay dos por cada plato**, uno para cada lado (***arriba*** y ***abajo***).
- **Track (Pista)**: Es un círculo concéntrico en la superficie de un plato, y hay dos (superficie superior e inferior) por plato. ***Contiene a los sectores***.
- **Sector**: Es la división de una pista en un arco, siendo la ***unidad de almacenamiento físico más pequeña***. Tradicionalmente, **512 bytes**, sin embargo, hoy también se utilizan sectores de **4096 bytes** (4KB - 4 Kilobytes).
- **Cylinder (Cilindro)**: Es el conjunto de ***pistas*** (por arriba y abajo), que se encuentran en la misma posición en todos los ***platos***. Es decir, un **cilindro** tendrá tantas ***pistas*** como ***cabezales***.

## Capacidad total de un disco duro
La capacidad (en **bytes**) de un disco duro, se calcula como _TC x TH x TST x TS_.
Siendo:
- **TC**: Total de ***cilindros***.
- **TH**: Total de ***cabezales***.
- **TST**: Total de ***sectores*** por ***track***.
- **TS**: Tamaño de un ***sector***.

## Rutinas de acceso a discos
El BIOS proporciona un conjunto de rutinas de acceso a discos que utilizan la familia de funciones **INT 0x13**. Estas funciones del BIOS son la **única forma** de acceder a los discos, hasta que se implemente un **controlador** adecuado.
Existen ***dos familias*** básicas llamadas **INT 0x13**. Una utiliza el direccionamiento **CHS** (***cylinder, head, sector***), y la otra el direccionamiento **LBA** (***Logical Block Addressing***).

### CHS
Este direccionamiento tiene limitaciones de direccionamiento, porque se diseñó en base a las limitaciones físicas de los discos duros iniciales. Normalmente, con un número fijo de bits para cilindros (10 bits -> 0 a 1023), cabezales (8 bits -> 0 a 255) y sectores (6 bits, con base 1 -> 1 a 63), solo se puede acceder a los primeros **8.455.716.864 bytes** (poco más de **8.45 GB**) del medio, como ***máximo***. Esto tiene que ver con los elementos por partición mostrados al inicio.
- **C (Cylinder)**: Cantidad de cilindros completos a saltar. Comenzando desde el **cilíndro** ***más interior***.
- **H (Head)**: Cantidad de cabezales completos a saltar. Comenzando desde el **plato** ***más alto***, y su superficie ***superior***.
- **S (Sector)**: Sectores de _offset_. Se añade a la suma de los ***sectores*** anteriores.

#### Ejemplo de direccionamiento CHS
Suponiendo que el fabricante especifica que el disco duro contiene en total **1020** ***cilindros***, **16** ***cabezales por cilindro***, **63** ***sectores por pista*** y con **512 bytes** cada uno...
La dirección **CHS(4, 3, 2)** = (4 * 16 * 63) + (3 * 63) + 2 = 4032 + 189 + 2 = 4223.

### LBA
EL direccionamiento LBA (Logical Block Address), identifica cada sector con un único número entero, empezando en 0. Es solo la representación lineal de la memoria en un disco duro. Dado esto, consiste en la ***dirección CHS*** - 1.

## Conversión de CHS a LBA y viceversa
Siendo:
- Hpc = Número de cabezas por cilindro.
- Spt = Número de sectores por pista.

### CHS a LBA
Con índices **C**, **H** base 0, y **S** base 1.
- **LBA** = (C x Hpc + H) x Spt + (S - 1).

Para visualizar mejor la fórmula, voy a usar el ejemplo de direccionamiento CHS, y lo transformaré a LBA...
Si **CHS(4, 3, 2)** = (4 * 16 * 63) + (3 * 63) + 2.
- Primero, factorizo por el término en común (Hpc). Con esto, queda que **CHS(4, 3, 2)** = 63 * ((4 * 16) + 3) + 2, lo que es igual a **Spt x ((C x Hpc) + H) + S**.
- Luego, es simplemente restarle la diferencia de base para el **sector** (porque comienza en 1, no en 0). Es decir, ***S - 1***.

Así, queda que **CHS = LBA + 1**. Es decir, **LBA = CHS - 1** (o ***LBA = Spt x ((C x Hpc) + H) + S - 1***).

### LBA a CHS
Dados **Hpc** y **Spt**.
- C = LBA // (Hpc x Spt).
- H = (LBA % (Hpc x Spt)) // Spt.
- S = ((LBA % (Hpc x Spt)) % Spt) + 1.

# Modos de operación de la arquitectura x86
## Real Mode (16 bits)
Modo original de los procesadores x86. Acceso a 1 MB de memoria. Permite acceso directo al hardware, 
pero carece de características modernas de protección de memoria.

### Direccionamiento de memoria en Real Mode
En este modo, hay poco más de 1 MB de memoria direccionable (incluyendo el **Área de Memoria Alta**), sin embargo, la cantidad utilizable es menor. El acceso a la memoria se realiza mediante **segmentación**, a través del sistema _segmento:offset_.

Hay seis registros de segmento de 16 bits: **CS** (_Code Segment_); **DS** (_Data Segment_); **ES** (_Extra Segment_); **SS** (_Stack Segment_); **FS**; y **GS**. Cuando se usan registros de segmento, las direcciones se indican mediante el _segmento:offset_ que dije hace un momento, siendo **segmento** un valor en un _registro de segmento_, y **offset** un valor en un _registro de dirección_. Los _segmentos_ y _offsets_ se relacionan con las **direcciones físicas** mediante la ecuación **Dirección física** = **Segmento** x 16 + **Offset**.

### Área de alta memoria
Si se establece **DS** (u otro _registro de segmento_) en **0xFFFF**, apunta a una dirección **16 bytes** por debajo de **1 MB**. Si se utiliza ese _registro de segmento_ como base, con un **offset** de **0x10** a **0xFFFF**, se puede acceder a direcciones de memoria física de **0x100000** a **0x10FFEF**. Esta área (de casi **64 kB**) por encima de **1 MB** se denomina ***Área de Memoria Alta*** en **Modo Real**. Para que esto funcione, se debe activar la línea de dirección **A20**.

#### Linea de dirección A20
La línea de dirección **A20** es la representación física del **bit** en la posición **21** de cualquier acceso a memoria. Su **importancia** radica en el **acceso completo de la memoria**, porque cuando está **desactivado**, no se puede acceder a más allá de **1 MB**.

Normalmente, esta línea viene desactivada para mantener la compatibilidad con sistemas más antiguos, pues, su uso lleva al ***reinicio*** de una dirección de memoria (es decir, pasa de 0xFFFF a 0x0000) al pasar de los **1 MB** disponibles al inicio. Desde el **Intel 80286**, los procesadores tienen más de 20 líneas de dirección en el ***bus***, por lo que se añadió un ***hardware gate*** a la línea **A20** y se dejó desactivada por defecto, simulando el ***reinicio*** de antaño.

#### Revisar A20
Para revisar si **A20** está activado o desactivado, se debe **comparar**, en el tiempo de **arranque (boot time)** y durante el **Modo Real**, el ***bootsector identifier*** **0xAA55** ubicado en la dirección **0x0000:0x7DFE**, con el valor **1 MiB** más arriba (dirección **0xFFFF:0x7E0E**). Si la comparación dicta que los valores son distintos, **A20** está activado, y en el caso contrario está **desactivado**. Para entenderlo mejor, la lógica de esta prueba es básicamente aprovechar esa vuelta o reinicio que da la dirección de memoria cuando **A20** está desactivado, es como dar una vuelta en 360° (se llega al mismo lugar). **Importante**, también se debe descartar la posibilidad de que si son iguales, sea simple casualidad, por lo que se lleva a cabo un "swapping" de la **firma mágica** y se vuelve a comparar.

#### Activar A20
Hay varias formas de activar **A20**, por ejemplo, a través de las **BIOS functions**; el **Controlador de teclado**; el método **Fast A20**, etc. En este caso, voy a mostrar el método del **INT 15** (BIOS), sacado de [Wiki OSDev - A20 Line](https://wiki.osdev.org/A20_Line#INT_15).
```assembly
;FASM
use16
mov     ax,2403h                ;--- A20-Gate Support ---
int     15h
jb      a20_ns                  ;INT 15h is not supported
cmp     ah,0
jnz     a20_ns                  ;INT 15h is not supported

mov     ax,2402h                ;--- A20-Gate Status ---
int     15h
jb      a20_failed              ;couldn't get status
cmp     ah,0
jnz     a20_failed              ;couldn't get status

cmp     al,1
jz      a20_activated           ;A20 is already activated

mov     ax,2401h                ;--- A20-Gate Activate ---
int     15h
jb      a20_failed              ;couldn't activate the gate
cmp     ah,0
jnz     a20_failed              ;couldn't activate the gate

a20_activated:                  ;go on
```

## Protected Mode (32 bits)
Introducido con el 80286. Acceso a más memoria y características de protección. **Protected Mode** soporta un espacio de direcciones de hasta **4 GB**. Incluye características como **segmentación** y **paginación**. Además, permite _multitarea_ y _protección de memoria_. El **Protected Mode** permite trabajar con ***direcciones de memoria virtuales***, y cada una con un máximo de **4 GB** de memoria direccionables. Además, restringe el conjunto de instrucciones disponibles mediante **Rings**.

En **Protected Mode** se desbloquea el verdadero potencial de la **CPU**. Sin embargo, en este modo no se pueden usar la mayoría de las **interrupciones BIOS**, porque éstas funcionan en **Real Mode**, a menos que se tenga el ***sub-modo V86*** (**Virtual 8086 Mode**).

### Entrar al Protected Mode
Antes de cambiar al **Protected Mode**, se debe hacer lo siguiente:
- Desactivar las interrupciones, incluyendo las **Non Maskable Interrupts (NMI)** (según el **Intel Developers Manual**).
- Activar la línea **A20**.
- Cargar la **GDT** con _descriptores de segmentos_ adecuados para **código**, **datos** y **stack**.

Luego, se debe colocar el **PE (Protection Enable)** bit del **CR0 (Control Register 0)** en 1 (ON). Para saber si la **CPU** está en **Protected Mode** o **Real Mode** se revisa el bit menos significativo del **CR0**.

### Virtual 8086 Mode
El **Virtual 8086 Mode** es un sub-modo del **Protected Mode**. Básicamente, es cuando la **CPU** (en **Protected Mode**) ejecuta una máquina en **Real Mode** de 16 bits ***emulada**.

#### Entrar al V86 Mode
La CPU se ejecuta en **V86 Mode** cuando el bit **VM** (bit 17 en el registro **EFLAGS**) está activado en el registro **EFLAGS**. Por ende, para entrar se debe activar ese bit. Una forma de modificar el registro **EFLAGS** es a través de las instrucciones ***PUSHF*** y ***POPF***, pero esas instrucciones son de **User Space** y no modifican **flags** de supervisor.

Por lo tanto, la única manera de activar el bit **VM** es usando la instrucción **IRET** (con CPL = 0), la cual se utiliza para retornar desde una interrupción. Cuando se llama a **IRET** con **VM** = 1 en el **stack EFLAGS**, el ***interrupt stack frame*** contendrá los segmentos **ES**, **DS**, **FS** y **GS**, de modo que puedan activarse antes de la entrada.

También, se puede usar una **Task Gate** para entrar al **Virtual Mode**, lo cual se hace mediante un cambio de tarea a una tarea **80386**, que establece el bit **VM** al cargar la imagen de **EFLAGS** desde el **Task State Segment** (TSS). Esto, permite activar los registros de segmento, sin embargo, es probable que esto no sea necesario, a menos que el sistema operativo esté utilizando ***hardware multitasking***.

#### Detectar V86 Mode
**EFLAGS.VM** jamás se empuja al **stack** si la tarea **V86** usa **PUSHFD**. Se debe comprobar si **CR0.PE** = 1 y asumir que se está en **V86 Mode**, si ese bit está activado.

## Long Mode (64 bits)
Introducido con el x86-64. Soporta dirección de memoria de 64 bits. Tiene compatibilidad con modos 
de 32 bits y 16 bits. Incluye 8 registros extra de propósito general (**r8 - r15**) y 8 registros multimedia (**xmm8 - xmm15**). 

# Interrupciones
## Maskable Interrupts

## Non Maskable Interrupts

# Interrupt Service Routines (ISR)
La arquitectura x86 es un sistema controlado por interrupciones. Los eventos externos desencadenan una interrupción, el flujo de control normal se interrumpe y se llama a una **ISR**. Estos eventos pueden ser activados por hardware o software. Las interrupciones controladas por software se activan mediante el **opcode** ***INT***. Para que el sistema sepa que **ISR** llamar cuando hay una interrupción, los ***offsets*** de las **ISR** se almacenan en la **Interrupt Descriptor Table (IDT)** cuando se está en ***Modo Protegido***, o en la **Interrupt Vector Table (IVT)** durante ***Real Mode***. Una **ISR** debe terminar con el **opcode** ***iret***.

## Llamada a los handlers
### x86
Cuando la CPU llama al **interrupt handler**, empuja los siguientes valores en el **stack** (en orden):
- **EFLAGS** -> **CS** -> **EIP**

El valor de **CS** se rellena con ***2 bytes*** para formar una **doubleword**. Si el **gate type** no es una **trap gate**, la CPU borra el ***interrupt flag***. Si la interrupción es una **excepción**, la CPU empuja un código de error en el **stack**, como una **doubleword**. La CPU cargará valor del selector de segmento del **IDT descriptor** asociado, en **CS**.

### x86-64
Cuando la CPU llama al **interrupt handler**, cambia el valor del registro **RSP** al valor especificado en el **Interrupt Service Thread (IST)**, y si no hay ninguno, el **stack** se mantiene igual. En el nuevo **stack**, la CPU inserta los siguientes valores (en orden):
- **SS:RSP** (**RSP** original) -> **RFLAGS** -> **CS** -> **RIP**

**CS** se rellena para formar una **quadword**. Si la interrupción se llama desde un **ring** distinto, **SS** se coloca en **0**, indicando un sector **nulo**. La CPU modifica el registro **RFLAGS**, colocando **TS** = **NT** = **RF** = **0**. Si la **gate type** es una **interrupt gate**, la CPU limpia el **interrupt flag**. Si la interrupción es una **excepción**, la CPU inserta un código de error en el **stack**, rellenado con bytes para hacer una **quadword**. La CPU cargará el valor del selector de segmento del **IDT descriptor** asociado, en **CS**, y comprobará que **CS** sea un selector de segmento válido.

# Interrupt Descriptor Table (IDT)
La **IDT** es una estructura de datos binaria específica de las arquitecturas ***IA-32*** y ***x86-64***. Es la contraparte en **Protected Mode** & **Long Mode** de la **Interrupt Vector Table (IVT)** en **Real Mode**, que indica a la ***CPU*** la ubicación de las **ISR** (una por vector de interrupción). Su estructura es similar a la de la **GDT**. Las entradas de la **IDT** se llaman "gates", y pueden haber ***Interrupt Gates***, ***Task Gates*** y ***Trap Gates***. Es crucial que, antes de implementar una **IDT**, se tenga una **GDT** funcionando como corresponde.

## IDTR (IDT Register)
Guarda la dirección de la **IDT**, y se carga usando la instrucción de **LIDT** de Assembly, la cual tiene por argumento un puntero hacia un estructura **IDT descriptor**.



# Interrupt Vector Table (IVT)
En la arquitectura x86, la **IVT** es una tabla que especifica las direcciones de los 256 ***interrupt handlers*** usados en **Real Mode** (16 bits). Típicamente, la **IVT** se localiza en 0000:0000H, y tiene un tamaño de 400H bytes (4 bytes por interrupción).

## Estructura de la IVT
Las entradas son consecutivas, lo cual significa que la primera entrada **apuntada** por la **IDTR** es el ***interrupt handler 0***, y las demás siguen en sucesión. El formato de una entrada es el siguiente:
```
 +-----------+-----------+
 |  Segment  |  Offset   |
 +-----------+-----------+
 4           2           0
```
Se puede ver que para obtener la dirección del **interrupt handler** que buscamos solo es **IDTR x 4**. Para cambiar un **interrupt handler**, lo único que se necesita es cambiar su dirección en la tabla.

## Distribución de las interrupciones de CPU
```
IVT Offset | INT #     | Description
-----------+-----------+-----------------------------------
0x0000     | 0x00      | Divide by 0
0x0004     | 0x01      | Reserved
0x0008     | 0x02      | NMI Interrupt
0x000C     | 0x03      | Breakpoint (INT3)
0x0010     | 0x04      | Overflow (INTO)
0x0014     | 0x05      | Bounds range exceeded (BOUND)
0x0018     | 0x06      | Invalid opcode (UD2)
0x001C     | 0x07      | Device not available (WAIT/FWAIT)
0x0020     | 0x08      | Double fault
0x0024     | 0x09      | Coprocessor segment overrun
0x0028     | 0x0A      | Invalid TSS
0x002C     | 0x0B      | Segment not present
0x0030     | 0x0C      | Stack-segment fault
0x0034     | 0x0D      | General protection fault
0x0038     | 0x0E      | Page fault
0x003C     | 0x0F      | Reserved
0x0040     | 0x10      | x87 FPU error
0x0044     | 0x11      | Alignment check
0x0048     | 0x12      | Machine check
0x004C     | 0x13      | SIMD Floating-Point Exception
0x00xx     | 0x14-0x1F | Reserved
0x0xxx     | 0x20-0xFF | User definable
```

## Distribución predeterminada de las interrupciones de hardware
- **Master 8259** (***Intel 8259A***), también conocido como **8259 PIC** (Controlador de Interrupciones Programable 8259).

Algunas interrupciones mapeadas por el **8259** se solapan con algunos **exception handlers** del procesador. Estas pueden remapearse a través de los **puertos de E/S** del **8259**.
```
IVT Offset | INT # | IRQ # | Description
-----------+-------+-------+--------------------------
0x0020     | 0x08  | 0     | PIT
0x0024     | 0x09  | 1     | Keyboard
0x0028     | 0x0A  | 2     | 8259A slave controller
0x002C     | 0x0B  | 3     | COM2 / COM4
0x0030     | 0x0C  | 4     | COM1 / COM3
0x0034     | 0x0D  | 5     | LPT2
0x0038     | 0x0E  | 6     | Floppy controller
0x003C     | 0x0F  | 7     | LPT1
```

- **Slave 8259**.
```
IVT Offset | INT # | IRQ # | Description
-----------+-------+-------+--------------------------
0x01C0     | 0x70  | 8     | RTC
0x01C4     | 0x71  | 9     | Unassigned
0x01C8     | 0x72  | 10    | Unassigned
0x01CC     | 0x73  | 11    | Unassigned
0x01D0     | 0x74  | 12    | Mouse controller
0x01D4     | 0x75  | 13    | Math coprocessor
0x01D8     | 0x76  | 14    | Hard disk controller 1
0x01DC     | 0x77  | 15    | Hard disk controller 2
```

# Global Descriptor Table (GBT)
La **GDT** es una estructura de datos binaria específica de las arquitecturas ***IA-32*** y ***x86-64***. Contiene entradas que informan a la CPU sobre segmentos de memoria.

# Stages del bootloader (convención)
## Stage 1 (MBR / VBR)
- Normaliza segmentos/stack.
- Guarda **DL** (número de disco del que se bootea).
- Se reubica para no pisarse al leer más sectores.
- Lee por **LBA** (Logical Block Addressing) (***INT 13h AH=42h***) un bloque contiguo (el **Stage 2**).
- Salta al **Stage 2**.

## Stage 1.5
- Rutinas de disco y **driver de FS** (FAT/Ext2,...) para encontrar archivos por nombre, no por LBA fijo.
- Puede preparar GDT y otras estructuras.

## Stage 2
- Consola, I/O básicas (VGA texto).
- Parsing y carga de **ELF64** del kernel.
- **Protected** --> **Long** Mode, mapeos iniciales, salto a ***entry*** del kernel.