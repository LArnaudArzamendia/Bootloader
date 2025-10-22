# BIOS
La BIOS (legacy) va al ***primer sector físico*** del disco booteable y copia los primeros **512 bytes** directo a la RAM. Convencionalmente, lo hace en la dirección física **0x7C00**. Aunque, esto puede variar, y es solamente una convención.
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
- Platter (Plato): Es un disco físico que almacena datos. Se encuentran apilados uno tras otro en una ruta vertical, para formar el disco duro. ***Contiene a las pistas***.
- **Head (Cabezal)**: Componente físico que lee y escribe los datos en las pistas. Se ubican al extremo del ***brazo***, que los posiciona sobre las partículas magnéticas. **Hay dos por cada plato**, uno para cada lado (***arriba*** y ***abajo***).
- **Track (Pista)**: Es un círculo concéntrico en la superficie de un plato, y hay dos (superficie superior e inferior) por plato. ***Contiene a los sectores***.
- **Sector**: Es la división de una pista en un arco, siendo la ***unidad de almacenamiento físico más pequeña***. Tradicionalmente, **512 bytes**, sin embargo, hoy también se utilizan sectores de **4096 bytes** (4KB - 4 Kilobytes).
- **Cylinder (Cilindro)**: Es el conjunto de ***pistas*** (por arriba y abajo), que se encuentran en la misma posición en todos los ***platos***. Es decir, un **cilindro** tendrá tantas ***pistas*** como ***cabezales***.
## Capacidad total de un disco duro
Siendo:
- **TC**: Total de ***cilindros***.
- **TH**: Total de ***cabezales***.
- **TST**: Total de ***sectores*** por ***track***.
- **TS**: Tamaño de un ***sector***.
La capacidad (en **bytes**) de un disco duro, se calcula como TC x TH x TST x TS.
## Rutinas de acceso a discos
El BIOS proporciona un conjunto de rutinas de acceso a discos que utilizan la familia de funciones **INT 0x13**. Estas funciones del BIOS son la **única forma** de acceder a los discos, hasta que se implemente un **controlador** adecuado.
Existen ***dos familias*** básicas llamadas **INT 0x13**. Una utiliza el direccionamiento **CHS** (***cylinder, head, sector***), y la otra el direccionamiento **LBA** (***Logical Block Addressing***).
### CHS
Este direccionamiento tiene limitaciones de direccionamiento, porque se diseñó en base a las limitaciones físicas de los discos duros iniciales. Normalmente, con un número fijo de bits para cilindros (10 bits -> 0 a 1023), cabezales (8 bits -> 0 a 255) y sectores (6 bits, con base 1 -> 1 a 63), solo se puede acceder a los primeros **8.455.716.864 bytes** (poco más de **8.45 GB**) del medio, como ***máximo***. Esto tiene que ver con los elementos por partición mostrados al inicio.
- **C (Cylinder)**: Cantidad de cilindros completos a saltar. Comenzando desde el **cilíndro** ***más interior***.
- **H (Head)**: Cantidad de cabezales completos a saltar. Comenzando desde el **plato** ***más alto***, y su superficie ***superior***.
- **S (Sector)**: Sectores "offset". Es el **sector** de la **pista** a leer.
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
Dado Hpc y Spt.
- C = LBA // (Hpc x Spt).
- H = (LBA % (Hpc x Spt)) // Spt.
- S = ((LBA % (Hpc x Spt)) % Spt) + 1.

# Modos de operación de la arquitectura x86
- Real Mode (16 bits): Modo original de los procesadores x86. Acceso a 1 MB de memoria. Permite acceso directo al hardware, 
pero carece de características modernas de protección de memoria.
- Protected Mode (32 bits): Introducido con el 80286. Acceso a más memoria y características de protección. Protected Mode, 
soporta un espacio de direcciones de hasta 4 GB. Incluye características como segmentación y paginación. Además, 
permite multitarea y protección de memoria.
- Long Mode (64 bits): Introducido con el x86-64. Soporta dirección de memoria de 64 bits. Tiene compatibilidad con modos 
de 32 bits y 16 bits.

# Interrupt Service Routines (ISR)
La arquitectura x86 es un sistema controlado por interrupciones. Los eventos externos desencadenan una interrupción, el flujo de control normal se interrumpe y se llama a una **ISR**. Estos eventos pueden ser activados por hardware o software. Las interrupciones controladas por software se activan mediante el **opcode** ***INT***. Para que el sistema sepa que **ISR** llamar cuando hay una interrupción, los ***offsets*** de las **ISR** se almacenan en la **Interrupt Descriptor Table (IDT)** cuando se está en ***Modo Protegido***, o en la **Interrupt Vector Table (IVT)** durante ***Real Mode***. Una **ISR** debe terminar con el **opcode** ***iret***.

# Interrupt Descriptor Table (IDT)
La **IDT** es una estructura de datos binaria específica de las arquitecturas ***IA-32*** y ***x86-64***. Es la contraparte en **Protected Mode** & **Long Mode** de la **Interrupt Vector Table (IVT)** en **Real Mode**, que indica a la ***CPU*** la ubicación de las **ISR** (una por vector de interrupción). Su estructura es similar a la de la **GDT**. Las entradas de la **IDT** se llaman "gates", y pueden haber ***Interrupt Gates***, ***Task Gates*** y ***Trap Gates***.

# Interrupt Vector Table (IVT)
En la arquitectura x86, la **IVT** es una tabla que especifica las direcciones de los 256 ***interrupt handlers*** usados en **Real Mode** (16 bits). Típicamente, la **IVT** se localiza en 0000:0000H, y tiene un tamaño de 400H bytes (4 bytes por interrupción).

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