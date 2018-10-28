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
Thus the vtable of cat class is:

| address | value | meaning|
|----|-----|----|
|0x400ac8|0x400ae0|The address of cat::sound(). If the vptr points here, it can call the parent implementation.|
|0x400ad0|0x400690|typeinfo, rtti metadat information, contains the hierarchy of inheritance.|

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

## Vtables and .rodata (With CFI, riscv64 as an example)


## Q & A

- How to index the member functions after interleaving the data?
- How to enforce CFI with these data?
- How to differentiate vcalls vs other indirect calls?
