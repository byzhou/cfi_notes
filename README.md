# CFI Notes

## Link Farms about Clang CFI

- [Video Tutorial](https://www.youtube.com/watch?v=QzJL-8WbpuU) explains how
vtable pointer works (when and where it points).
- [Layout of Vtables](https://shaharmike.com/cpp/vtable-part1/) explains how
the vtable laout works (Address, value, and meaning).
- [Layout of
ELF](https://www.martinkysel.com/demystifying-virtual-tables-in-c-part-3-virtual-tables/)
explains how to read vtable metadata information in the binary.
- [Type Metadata](https://llvm.org/docs/TypeMetadata.html) records how the type
metadata information is stored in the binary.
- [Control Flow Integrity Clang
Design](https://clang.llvm.org/docs/ControlFlowIntegrityDesign.html) explains
how the memory are laid out in the .rodata section.
- [Paper](https://cseweb.ucsd.edu/~lerner/papers/ivtbl-ndss16.pdf) explains how
the .rodata are organized: pre-order hierarchy and interleaving data.

- [Backend Pass Tutorial](http://www.kharghoshal.xyz/blog/writing-machinefunctionpass)
explains how to write a backend pass. super useful, stupid llvm.org only has tutorials
for writing one entire backend. 

## Vtables and .rodata (Without CFI, x86 as an example)

The metadata information of all the class definitions are stored inside the
*.rodata*.  CPP uses name mangling for differentiating functions, members of
classes in different namespaces.  It usually append the original name after
prefixes and append suffixes after the original names.  For the metadata
information, all the names in the metadata are mangled by the compiler.
However, they mangling naming follows a consistent rule.  The metadata consists
of 3 parts: 1) vtable, 2) typeinfo name, and 3) typeinfo. The corresponding
mangling prefixes are 1)\_ZTV, 2)\_ZTS 3)\_ZTI.

| Class Name | Vtable | Typeinfo Name | Typeinfo |
|----------|------|-----|-----|
|cat | \_ZTV3cat| \_ZTS3cat |\_ZTI3cat|

Dynamically determine the vtable information:
```bash
gdb ./your-binary-name
> info symbol 0xaddress-of-symbol
```

In order to display the vtable information statically, you can use objdump:
```bash
(cross-compiler-triple)objdump -d -j .rodata your-binary-name

# Here are the rodata information.

Disassembly of section .rodata:

0000000000400a90 <_IO_stdin_used>:
  400a90:	01 00 02 00 6d 65 6f 77 77 77 0a 00 70 75 66 66     ....meowww..puff
  400aa0:	66 0a 00 49 20 61 6d 20 73 6d 61 72 74 0a 00 49     f..I am smart..I
  400ab0:	20 61 6d 20 63 75 74 65 0a 00 00 00 00 00 00 00      am cute........

0000000000400ac0 <_ZTV3cat>:
	...
  400ac8:	e0 0a 40 00 00 00 00 00 90 06 40 00 00 00 00 00     ..@.......@.....

0000000000400ad8 <_ZTS3cat>:
  400ad8:	33 63 61 74 00 00 00 00                             3cat....

0000000000400ae0 <_ZTI3cat>:
  400ae0:	a8 1d 60 00 00 00 00 00 d8 0a 40 00 00 00 00 00     ..`.......@.....
  400af0:	c0 0b 40 00 00 00 00 00                             ..@.....
```
Thus the vtable of cat class is (Runtime typeinfo, RTTI) :

| Address | Value | meaning|
|----|-----|----|
|0x400ac8|0x400ae0|The address of cat::sound(). If the vptr points here, it can call the parent implementation.|
|0x400ad0|0x400690|typeinfo, RTTI metadat information, contains the address of the parent's RTTI.|

## Vtables and .rodata (Without CFI, riscv64 as an example)
Here is an example of how the vtables and .rodata laid out in the binary.
```cpp
#include <stdio.h>
class animal {
    public:
    virtual void sound(){};
};
class cat:public animal{
    public:
    virtual void sound();
};
class dog:public animal{
    public:
    virtual void sound();
};
class retriever:public dog{
    public:
    virtual void sound();
};
class doge:public dog{
    public:
    virtual void sound();
};

void cat::sound(){
    printf ("meowww\n");
}
void dog::sound(){
    printf ("pufff\n");
}
void retriever::sound(){
    printf ("I am smart\n");
}
void doge::sound(){
    printf ("I am cute\n");
}
```
The inheritance tree is as follows:
```
anmial --- dog  --- retriever
       |        |
       |        --- doge
       |
       --- cat
```
By dumping the .rodata information from the binary, we can get:

```bash
# riscv64-unknown-linux-gnu-objdump -d -j .rodata helloworld 

helloworld:     file format elf64-littleriscv


Disassembly of section .rodata:

0000000000010ca0 <_ZTV3cat-0x30>:
   10ca0:	656d                	lui	a0,0x1b
   10ca2:	7777776f          	jal	a4,88c18 <__global_pointer$+0x753e0>
   10ca6:	000a                	c.slli	zero,0x2
   10ca8:	7570                	ld	a2,232(a0)
   10caa:	6666                	ld	a2,88(sp)
   10cac:	0a66                	slli	s4,s4,0x19
   10cae:	4900                	lw	s0,16(a0)
   10cb0:	6120                	ld	s0,64(a0)
   10cb2:	206d                	0x206d
   10cb4:	72616d73          	csrrsi	s10,0x726,2
   10cb8:	0a74                	addi	a3,sp,284
   10cba:	4900                	lw	s0,16(a0)
   10cbc:	6120                	ld	s0,64(a0)
   10cbe:	206d                	0x206d
   10cc0:	65747563          	bleu	s7,s0,1130a <__FRAME_END__+0x41a>
   10cc4:	000a                	c.slli	zero,0x2
	...

0000000000010cd0 <_ZTV3cat>:
	...
   10cd8:	0cf0 0001 0000 0000 06a0 0001 0000 0000     ................

0000000000010ce8 <_ZTS3cat>:
   10ce8:	6333 7461 0000 0000                         3cat....

0000000000010cf0 <_ZTI3cat>:
   10cf0:	2db8 0001 0000 0000 0ce8 0001 0000 0000     .-..............
   10d00:	0dd0 0001 0000 0000                         ........
```
Thus, we can infer from the snippet dissambled binary:

| Address | Value | Meaning|
| -----   | ----- | ------ |
| 0x10cd0 | 0x0 | Zero padding |
| 0x10cd8 | 0x10cf0 | RTTI information of the cat class, ZTV as prefix for vtable | 
| 0x10ce0 | 0x106a0 | cat::sound() function address. vptr points here once cat::sound() is called.|
| 0x10ce8 | 0x74616333 | ASCII for "3cat"|
| 0x10cf0 | 0x12db8 | cxxabi for class type info, used for getting the class names and metadata info in sources.|
| 0x10cf8 | 0x10ce8 | address of ASCII for "3cat" | 
| 0x10d00 | 0x10dd0 | RTTI information of the parent class animal | 

To get the cxxabi class type info function, you can use the following command ( not relevant in this context, but good to know):

```bash
$ riscv64-unknown-linux-gnu-objdump -d -j .data.rel.ro helloworld

helloworld:     file format elf64-littleriscv


Disassembly of section .data.rel.ro:

0000000000012d50 <_ZTVN10__cxxabiv117__class_type_infoE@@CXXABI_1.3>:
	...

0000000000012da8 <_ZTVN10__cxxabiv120__si_class_type_infoE@@CXXABI_1.3>:
```

## Vtables and .rodata (With CFI, riscv64 as an example)
The same
[example](https://github.com/byzhou/cfi_notes#vtables-and-rodata-without-cfi-riscv64-as-an-example)
as the last section, with *-flto -fvisibility=hidden -fsanitize=cfi* enabled
during clang compilation.

```bash
# riscv64-unknown-linux-gnu-objdump -d -j .rodata helloworld 

helloworld:     file format elf64-littleriscv


Disassembly of section .rodata:

0000000000010c70 <_ZTI4doge-0x30>:
   10c70:	656d                	lui	a0,0x1b
   10c72:	7777776f          	jal	a4,88be8 <__global_pointer$+0x753a8>
   10c76:	000a                	c.slli	zero,0x2
   10c78:	7570                	ld	a2,232(a0)
   10c7a:	6666                	ld	a2,88(sp)
   10c7c:	0a66                	slli	s4,s4,0x19
   10c7e:	4900                	lw	s0,16(a0)
   10c80:	6120                	ld	s0,64(a0)
   10c82:	206d                	0x206d
   10c84:	72616d73          	csrrsi	s10,0x726,2
   10c88:	0a74                	addi	a3,sp,284
   10c8a:	4900                	lw	s0,16(a0)
   10c8c:	6120                	ld	s0,64(a0)
   10c8e:	206d                	0x206d
   10c90:	65747563          	bleu	s7,s0,112da <__FRAME_END__+0x476>
   10c94:	000a                	c.slli	zero,0x2
	...

0000000000010ca0 <_ZTI4doge>:
   10ca0:	2db8 0001 0000 0000 0cb8 0001 0000 0000     .-..............
   10cb0:	0cc0 0001 0000 0000                         ........

0000000000010cb8 <_ZTS4doge>:
   10cb8:	6434 676f 0065 0000                         4doge...

0000000000010cc0 <_ZTI3dog>:
   10cc0:	2db8 0001 0000 0000 0cd8 0001 0000 0000     .-..............
   10cd0:	0ce0 0001 0000 0000                         ........

0000000000010cd8 <_ZTS3dog>:
   10cd8:	6433 676f 0000 0000                         3dog....

0000000000010ce0 <_ZTI6animal>:
   10ce0:	2d60 0001 0000 0000 0cf0 0001 0000 0000     `-..............

0000000000010cf0 <_ZTS6animal>:
   10cf0:	6136 696e 616d 006c 0000 0000 0000 0000     6animal.........

0000000000010d00 <_ZTI9retriever>:
   10d00:	2db8 0001 0000 0000 0d18 0001 0000 0000     .-..............
   10d10:	0cc0 0001 0000 0000                         ........

0000000000010d18 <_ZTS9retriever>:
   10d18:	7239 7465 6972 7665 7265 0000 0000 0000     9retriever......
	...

0000000000010d30 <_ZTI3cat>:
   10d30:	2db8 0001 0000 0000 0d48 0001 0000 0000     .-......H.......
   10d40:	0ce0 0001 0000 0000                         ........

0000000000010d48 <_ZTS3cat>:
   10d48:	6333 7461 0000 0000 0000 0000 0000 0000     3cat............
   10d58:	0ca0 0001 0000 0000 0788 0001 0000 0000     ................
	...
   10d78:	0d00 0001 0000 0000 0750 0001 0000 0000     ........P.......
	...
   10d98:	0cc0 0001 0000 0000 0718 0001 0000 0000     ................
	...
   10db8:	0ce0 0001 0000 0000 0bf0 0001 0000 0000     ................
	...
   10dd8:	0d30 0001 0000 0000 06e0 0001 0000 0000     0...............
```

Compared to the one without CFI, the .rodata has reformatted all the
information.  Instead of laying out the ZTV, ZTI, and ZTS metadata information,
the .rodata is organized to lay out typeinfo + typeinfo name upfront (before
_0x10d50_), and then lay out the vtable information interleaved together (after
_0x10d50_). Here are the explanations on the vtable information:

| Address | Value | Meaning |
| -----   | ----- | ------ |
| 0x10d58 | 0x10ca0 | the address of rtti doge|  
| 0x10d60 | 0x10788 | the address of doge::sound() | 
| 0x10d78 | 0x10d00 | the address of rtti retriever|  
| 0x10d78 | 0x10750 | the address of retriever::sound() | 
| 0x10d98 | 0x10cc0 | the address of rtti dog |  
| 0x10d98 | 0x10718 | the address of dog::sound() | 
| 0x10db8 | 0x10ce0 | the address of rtti animal |   
| 0x10db8 | 0x10bf0 | the address of animal::sound() | 
| 0x10dd8 | 0x10d30 | the address of rtti cat | 
| 0x10dd8 | 0x106e0 | the address of cat::sound () | 

In the [paper](https://cseweb.ucsd.edu/~lerner/papers/ivtbl-ndss16.pdf), the
authors mentioned that their interleaved layout of vtables, can be enforced
using intel [MPX](https://intel-mpx.github.io/code/submission.pdf). In the
brief idea, the intel MPX introduces new ISAs to check the memory boundaries
against certain pre-defined values.
[MPX](https://intel-mpx.github.io/code/submission.pdf) explains that the
current implementation of compiler does not work well with the MPX, in the
sense that the overhead of memory checks was 4x. In addition to the slowdowns,
8-13% of the programs did not run correctly without substantial code changes,
and 18% required non-intrusive manual changes.

## Software Enforcement

Software Enforcement uses instructions to check if the target address is aligned and
within certain range.

```
assert (a <= $vptr <= b) and ($vptr & 0x1f == 0)
```
In one real case, the above code is optimized into ''srli'', ''slli'' and
several other instructions.  Warning: the following asm is not from the
previous
[example](https://github.com/byzhou/cfi_notes#vtables-and-rodata-without-cfi-riscv64-as-an-example).
This piece of code is used to show how CFI is enfoced.

```bash
   10678:       fd043503                ld      a0,-48(s0)
   1067c:       00053583                ld      a1,0(a0)
   10680:       00055637                lui     a2,0x55
   10684:       fc060613                addi    a2,a2,-64 # 54fc0 <_ZTS3cat+0x18>
   10688:       40c58633                sub     a2,a1,a2
   1068c:       00565693                srli    a3,a2,0x5
   10690:       03b61613                slli    a2,a2,0x3b
   10694:       00c6e633                or      a2,a3,a2
   10698:       00563613                sltiu   a2,a2,5
   1069c:       00061a63                bnez    a2,106b0 <main+0x80>
   106a0:       0040006f                j       106a4 <main+0x74>
   106a4:       00010537                lui     a0,0x10
   106a8:       20050513                addi    a0,a0,512 # 10200 <abort>
   106ac:       000500e7                jalr    a0
   106b0:       0005b583                ld      a1,0(a1)
   106b4:       000580e7                jalr    a1
```
## Build Chromium

### x86 default
To build chromium in linux, use the following commands. 

```bash
# download all the tools to build chromium
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
cd depot_tools
export PATH=$PATH:`pwd`
cd ..
mkdir -p chromium-build
cd chromium-build
# download all the necessary dependencies
fetch --nohooks chromium
cd src
# download additional dependencies
gclient runhooks
# configure ninja
gn gen out/Default
# you can use gn to configure ninja build dependencies, cross-compiler triple
# are specified for ''gn''. You can use ''gn help'' to specify compiler, flags
# used for building with ninja
autoninja -C out/Default chrome
# location of chrome
out/Default/chrome
```
To use chrome to dump website to stdout:

```bash
google-chrome -headless -no-sandbox -incognito  --app=https://google.com
```
-headless means that do not display GUI

-no-sandbox will be a security issue 

-dump-dom means that to print document.body.innerHTML to stdout

## Build Apache

### x86 default

Prerequisites of apache build.
```bash
sudo apt install libaprutil1 libaprutil1-dev libapr1 libapr1-dev
```
[Apache HTTP Server](http://httpd.apache.org/docs/2.4/install.html) explains
how apache server is downloaded and installed. To make sure that it is
installed properly, you need to satisfy the prerequisites. The compilation
informations are configured using environmental variables, similar to any other
linux programs.

## RISCV static

Configuration error in Apache:
```
configure: error: Size of "void *" is less than size of "long"
# change the configure file
# if test "$ap_cv_void_ptr_lt_long" = "yes"; then
#     as_fn_error $? "Size of \"void *\" is less than size of \"long\"" "$LINENO" 5
# 
# change to 
#
# if test "$ap_cv_void_ptr_lt_long" != "yes"; then
#     as_fn_error $? "Size of \"void *\" is less than size of \"long\"" "$LINENO" 5

# cannot remove libtoolT: No such file or directory
# chang $RM "$cfgfile" to $RM -f "$cfgfile"
```

## Q & A

- How to index the member functions after interleaving the data?
- How to enforce CFI with these data?
- How to differentiate vcalls vs other indirect calls?
- How to handle devirtualization in single member function classes?
- How to handle singleton?

## Problems Remained

- How to differentiate between virtual calls and other indirect calls?
