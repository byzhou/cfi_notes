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

## Vtables and .rodata

The metadata information of all the class definitions are stored inside the
*.rodata*.  CPP uses name mangling for differentiating functions, members of
classes in different namespaces.  It usually append the original name after
prefixes and append suffixes after the original names.  For the metadata
information, all the names in the metadata are mangled by the compiler.
However, they mangling naming follows a consistent rule.  The metadata consists
of 3 parts: 1) vtable, 2) typeinfo name, and 3) typeinfo. The corresponding
mangling prefixes are 1)\_ZTV, 2)\_ZTS 3)\_ZTI.

## Q & A

- How to index the member functions after interleaving the data?
- How to enforce CFI with these data?
- How to differentiate vcalls vs other indirect calls?
