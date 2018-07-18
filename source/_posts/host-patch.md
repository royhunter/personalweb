title: Linker for hotpatch
date: 2016-08-26 18:37:12
tags: PROGRAMME
categories: PROGRAMME
---
一个小型链接器的实现，主要功能为obj文件符号重定位，可用作内存载入ELF的hotpatch。以下为一些实现的原理和思路
<!--more--> 

## Why hotpatch?
- Compile entire product image/package costly
    - Just modify one line of one function
    - Accelerate dev’s debugging 

- Reboot system to upgrade will suspend traffic
    - Apply and valid instantly

- Only support memory load ELF currently


## Procedure(offline)
- Compile .c to .o with functions which need to be patched

- Link .o(obj) file by executable file
    - ELF analyze
    - Symbol relocate 
    - Instruction modify
    - Section layout(.text/.data/.bss/.rodata)
    - output image

- Pack image and patch info to a single package
    - Comply with regular patch format(user-defined)

## Procedure(online)
- Load patch package into box
- Parser patch (patch format) 
- Alloc patch memory area and copy image body
- Drop Process into idle state
- Modify two instructions of entrance of patched function
    - j target
    - nop
- Invalid ICACHE and DCACHE
- Restore into running state

## Compile obj
- test.c

```bash
xxx-gcc -mlong-calls -Wno-pointer-sign -Wno-error=address …… –c test.c
```
- test.o

## ELF background
- Header
- Section
- Symbol table
- String table

### ELF Header
```bash
readelf -h test_mips.o
```

```c
#define EI_NIDENT (16)

typedef struct
{
  unsigned char e_ident[EI_NIDENT];     /* Magic number and other info */
  Elf32_Half    e_type;                 /* Object file type */
  Elf32_Half    e_machine;              /* Architecture */
  Elf32_Word    e_version;              /* Object file version */
  Elf32_Addr    e_entry;                /* Entry point virtual address */
  Elf32_Off     e_phoff;                /* Program header table file offset */
  Elf32_Off     e_shoff;                /* Section header table file offset */
  Elf32_Word    e_flags;                /* Processor-specific flags */
  Elf32_Half    e_ehsize;               /* ELF header size in bytes */
  Elf32_Half    e_phentsize;            /* Program header table entry size */
  Elf32_Half    e_phnum;                /* Program header table entry count */
  Elf32_Half    e_shentsize;            /* Section header table entry size */
  Elf32_Half    e_shnum;                /* Section header table entry count */
  Elf32_Half    e_shstrndx;             /* Section header string table index */
} Elf32_Ehdr;
```

![http://7j1zal.com1.z0.glb.clouddn.com/ELF-Section.PNG](http://7j1zal.com1.z0.glb.clouddn.com/ELF-Section.PNG)

### Section Header
```bash
readelf -S test_mips.o
readelf -S  b.o
```

```c
typedef struct
{
  Elf32_Word    sh_name;                /* Section name (string tbl index) */
  Elf32_Word    sh_type;                /* Section type */
  Elf32_Word    sh_flags;               /* Section flags */
  Elf32_Addr    sh_addr;                /* Section virtual addr at execution */
  Elf32_Off     sh_offset;              /* Section file offset */
  Elf32_Word    sh_size;                /* Section size in bytes */
  Elf32_Word    sh_link;                /* Link to another section */
  Elf32_Word    sh_info;                /* Additional section information */
  Elf32_Word    sh_addralign;           /* Section alignment */
  Elf32_Word    sh_entsize;             /* Entry size if section holds table */
} Elf32_Shdr;
```

#### sh_type
```c
#define SHT_NULL          0          Section header table entry unused 
#define SHT_PROGBITS      1          Program data 
#define SHT_SYMTAB        2          Symbol table 
#define SHT_STRTAB        3          String table 
#define SHT_RELA          4          Relocation entries with addends 
#define SHT_HASH          5          Symbol hash table 
#define SHT_DYNAMIC       6          Dynamic linking information 
#define SHT_NOTE          7          Notes 
#define SHT_NOBITS        8          Program space with no data (bss) 
#define SHT_REL           9          Relocation entries, no addends 
```

#### sh_flags
```c
#define SHF_WRITE            (1 << 0)      Writable 
#define SHF_ALLOC            (1 << 1)      Occupies memory during execution 
#define SHF_EXECINSTR        (1 << 2)      Executable 
#define SHF_MERGE            (1 << 4)      Might be merged 
#define SHF_STRINGS          (1 << 5)      Contains nul-terminated strings
#define SHF_INFO_LINK        (1 << 6)      `sh_info' contains SHT index 
```

### .txt
```bash
xxx-objdump -d test.o
```

### .strtab 字符串表
ELF文件中用到很多字符串，比如段名、变量名。因为字符串的长度往往是不固定的，所以用固定的结构来表示它比较困难。一种常见的做法就是把字符串集中起来存放到一个表，然后使用字符串在表中的偏移来引用。
![http://7j1zal.com1.z0.glb.clouddn.com/ELF-Stringtable.PNG](http://7j1zal.com1.z0.glb.clouddn.com/ELF-Stringtable.PNG)

### .symtab 符号表
```c
typedef struct
{
  Elf32_Word    st_name;                /* Symbol name (string tbl index) */
  Elf32_Addr    st_value;               /* Symbol value */
  Elf32_Word    st_size;                /* Symbol size */
  unsigned char st_info;                /* Symbol type and binding */
  unsigned char st_other;               /* Symbol visibility */
  Elf32_Section st_shndx;               /* Section index */
} Elf32_Sym;
```
- sh_value:符号的地址
- sh_shndx: 符号所在的段在段表中的下标

#### st_info
```c
#define ELF32_ST_BIND(val)              (((unsigned char) (val)) >> 4)
#define ELF32_ST_TYPE(val)              ((val) & 0xf)
#define ELF32_ST_INFO(bind, type)       (((bind) << 4) + ((type) & 0xf))
```
![http://7j1zal.com1.z0.glb.clouddn.com/ELF_SymInfo.PNG](http://7j1zal.com1.z0.glb.clouddn.com/ELF_SymInfo.PNG)

### 强/弱符号
- 函数和初始化了的全局变量为强符号
- 未初始化的全局变量为弱符号
- __attribute__((weak))
- 强弱针对定义,而不针对声明

#### 强/弱符号的链接选择
1. 不允许多次定义强符号 
    “multiple definition of “xxx””
2. 如果一个符号在某个.o中是强符号,在其它.o中是弱符号,那么选择强符号
3. 如果一个符号在所有.o中是弱符号,选择占用空间最大的一个
```c
int global;     
double global;
```

#### case
```c
__attribute__ ((weak)) void foo();
 
int main()
{
	if (foo)
     foo();
}
```

### COMMON块
- 未初始化的全局变量
```bash
 23: 00000004     4 OBJECT  GLOBAL DEFAULT  COM global_undef
```
- Why not BSS?
- obj_allocate_commons
    - Find the bss section, if not exist, create one
    - Allocate the COMMONS
    - Allocate space for BSS

### .rel.text 重定位表
```bash
readelf -r test_mips.o
readelf -r b.o
```
```c
typedef struct
{
  Elf32_Addr    r_offset;               /* Address */
  Elf32_Word    r_info;                 /* Relocation type and symbol index */
  Elf32_Sword   r_addend;               /* Addend */
} Elf32_Rela;
```
- r_offset: 重定位入口的偏移,重定位入口所要修正的位置的第一个字节相对于段起始的偏移
- r_info: 低8位表示重定位入口的类型，高24位表示重定位入口的符号在符号表中的下标
- r_addend: 保存在被修正位置的值

### 重定位入口的类型
```c
#define R_MIPS_NONE             0          No reloc 
#define R_MIPS_16               1              Direct 16 bit 
#define R_MIPS_32               2              Direct 32 bit 
#define R_MIPS_REL32            3          PC relative 32 bit
#define R_MIPS_26               4              Direct 26 bit shifted 
#define R_MIPS_HI16             5            High 16 bit 
#define R_MIPS_LO16             6           Low 16 bit 
#define R_MIPS_GPREL16          7       GP relative 16 bit 
#define R_MIPS_LITERAL          8         16 bit literal entry 
#define R_MIPS_GOT16            9        16 bit GOT entry 
#define R_MIPS_PC16             10        PC relative 16 bit 
#define R_MIPS_CALL16           11      16 bit GOT entry for function 
#define R_MIPS_GPREL32          12     GP relative 32 bit 
```

### -mlong-calls
```c
gcc -mlong-calls -Wno-pointer-sign -Wno-error=address …… –c test.c
```

#### R_MIPS_26
```c
00000000 <function_handler>:
   0:   27bdffe8        addiu   sp,sp,-24
   4:   afbf0010        sw      ra,16(sp)
   8:   0c000000        jal     0 <function_handler>
   c:   00a02021        move    a0,a1
  10:   8fbf0010        lw      ra,16(sp)
  14:   00001021        move    v0,zero
  18:   03e00008        jr      ra
  1c:   27bd0018        addiu   sp,sp,24
```

#### R_MIPS_HI16/ R_MIPS_LO16
```c
00000000 <function_handler>:
   0:   27bdffe8        addiu   sp,sp,-24
   4:   afbf0010        sw      ra,16(sp) function_handler(m);
   8:   3c020000        lui     v0,0x0                       0x3c021813
   c:   24420000        addiu   v0,v0,0                  0x2442aa44
  10:   0040f809        jalr    v0
  14:   00a02021        move    a0,a1
  18:   8fbf0010        lw      ra,16(sp)
  1c:   00001021        move    v0,zero
  20:   03e00008        jr      ra
  24:   27bd0018        addiu   sp,sp,24
```

### jal/jalr
![http://7j1zal.com1.z0.glb.clouddn.com/MIPS-JUMP.PNG](http://7j1zal.com1.z0.glb.clouddn.com/MIPS-JUMP.PNG)

### Variable relocate
```c
static_yyy = 0;
2c    3c020000        lui     v0,0x0                      0x3c025000
30:   ac400004        sw      zero,4(v0)                  0xac400094   
static_zzz = 0;
34:   3c030000        lui     v1,0x0                      0x3c035000
38:   ac600000        sw      zero,0(v1)                  0xac600090

```

## Advanced topic
1. Dynamic Link
2. –fPIC, Position-independent Code
3. GOT, Global Offset Table 
4. PLT, Procedure  Linkage Table

## Resource
- https://github.com/royhunter/hotpatch_linker 
- 程序员的自我修养—链接 装载与库
- insmod.c
    - modutils 2.4.27
    - ftp://ftp.kernel.org/pub/linux/utils/kernel/modutils/v2.4/ 
- readelf.c
    - binutils 
    - https://www.gnu.org/software/binutils/ 
    - http://ftp.gnu.org/gnu/binutils/ 
