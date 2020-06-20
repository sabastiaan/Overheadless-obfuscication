---
layout: post
title:  "Binary overlapping?"
date:   2020-06-20 23:09:46 +0200
categories: x86 bin
---



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
0f 0d c0        NOP EAX, EAX
90              NOP
90              NOP
31  ff          XOR EDI, EDI
```

If executed, it will first read the byte 0f, this doesn't mean anything yet. So we will parse a second byte, c0. Still nothing, then we read d0 and we get a specified instruction. In this case NOP EAX, EAX. Best described as "No OPeration executed with register EAX and EAX".

Afterwards we read two NOP's and a XOR insruction that XOR's both it's inputs. Hence zero'ing out all content!.

What this program does is not really that interesting. Afterall it does nothing for a bit and then only zero's out an register. But after 3 years this is the most beautifull program I could think of.

First things first, why do we execute the same operation for the byte sequence `0f 0d c0` as for `90`. They share no bytes so what is up with this?

That is because if we look up `0f 0d c0` in the instruction manual, we can see that it is an instruction for prefechting hints. So it should not effect state on the cpu (or at least observable to us). And because of this the dissasmbler in which I generated this output decided it was equivelent to a NOP, and should be stated as such.

However this above piece of code is only disassembled like that because I throw it inside a proper disassembler, let's take a look at the output of objdump.

```
0f                      prefetch (bad)
0d c0 90 90 31          or     $0x319090c0,%eax
....
```

TThree interstings things happend here, first of all it didn't decode it correctly. Which isn't to uncommon even in mainstream dissasemblers  [0]. 

The second interesting that happend we suddenly obtained a totally new instruction that we have not seen before by starting execution/parsing one byte later!

But most importantly of all, __what is the next executed instruction__?
We know it is should be the next byte in the stream, so looking at the first output, it should start with _ff__.

If we would continue this, we would have to completely indepedented programs inside the same original binary sequence!

# Usecases

This technique has 2 very interesting use cases. The original reason how I came up with this idea, obfuscation. 
Notice that the second output, the output of the dissasembler objdump, can now theoritically be completely different from what another actor, like your CPU, could read. Implying that maleware can be completely hidden from maleware analysis or automatic analysis tools without any overhead, by exploiting a single parsing error. Which can be guarteed against most common techniques (this will the topic of another blog).

The second and probably more exciting use case is performance.
Above we didn't really gave that much thought on what the programs are. Let us consider a program P. If we take the second half of P, and we were magically able to alter the first half of P such that it's functionallity would be retained, and that if you would start parsing from the second byte it would execute the second half. Then we have effectively halfed our program size! And we could do this recursively until we have reducecd it by a massive factor.


# Required properties for overlapping
If we consider binary sequence, we say they are binary overlapping, if and only if, we can obtain a program P and P' such that
1. To execute P, we start executing at a different byte in the byte sequence than for P'
2. For each instruction in P, no instruction ends with the same byte index in the seqeunce as any other in P'

The first property is to ensure we talk about programs that we about programs that are obtained by starting at a different location than another.

The second property is there to stop them from converging. If this property wasn't satisfied, any 2 different programs would converge into the same from that point onwards, example:

```
ff c5                   inc    ebp
fc                      cld
77 90                   ja     0xffffffffffffff95
90                      nop             <- same as the second stream
....
```

```
c5 fc 77                vzeroall
90                      nop
90                      nop             <- same as the first stream
....
```


# generating this


*take this definition as wide as you wish, it can be desired, undesired, valid or unvalid. You could techniqually have a desired unvalid program by catching UD exceptions. I just thought it would be helpfull to include some notion of correctness for sanities sake. That's also why I choose the wording sementacily valid. Since I sure as hell can't define what __that__ is supposed to mean




[0] Paleari, R., Martignoni, L., Fresi Roglia, G., & Bruschi, D. (2010, July). N-version disassembly: differential testing of x86 disassemblers. In Proceedings of the 19th international symposium on Software testing and analysis (pp. 265-274).



