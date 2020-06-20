# Binary overlapping

When an x86 CPU executes a program, a pointer points towards a seqeunce of bytesf
Which will get read byte for byte untill the CPU recognizes the instruction and it will execute it.
After execution is done, it will read the next byte until it recognizes it as an instruction repeating this process.
The CPU's today will not strictly adhere this model anymore and it might execute/parse instructions in advance under the hood already, 
but it still has to guarentee this model works. We will also only consider this model because other software like disassemblers typically work this way.

In x86, instructions are by definition anywhere between 1-15 bytes long, they follow a *really* complex encoding scheme which we will ignore for now. 
Furthermore the word "instruction" is a little bit ambigues, since it can refer to 
1. the mnemnoic, i.e ADD 
2. the assembler form: i.e ADD rax, 2
3. the binary encoding, i.e for ADD that might be 0x10

The last final thing you should know about is that instructions can have multiple binary encodings.


So lets dive into the good stuff!

Consider the binary following binary sequence

```

```
