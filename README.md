# CFI Notes

## Link Farms about Clang CFI

- [Layout of ELF](https://www.martinkysel.com/demystifying-virtual-tables-in-c-part-3-virtual-tables/) explains how to read vtable metadata information in the binary.
- [Type Metadata](https://llvm.org/docs/TypeMetadata.html) records how the type metadata information is stored in the binary.
- [Control Flow Integrity Clang Design](https://clang.llvm.org/docs/ControlFlowIntegrityDesign.html) explains how the memory are laid out in the .rodata section.
- [Paper](https://cseweb.ucsd.edu/~lerner/papers/ivtbl-ndss16.pdf) explains how the .rodata are organized: pre-order hierarchy and interleaving data.


## Q & A

- How to index the member functions after interleaving the data?
- How to enforce CFI with these data?
- How to differentiate vcalls vs other indirect calls?
