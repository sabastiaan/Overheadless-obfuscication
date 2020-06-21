---
layout: post
title:  "Binary overlapping"
date:   2020-06-20 23:09:46 +0200
categories: x86 bin
---



# Binary overlapping
x86 programs underpin a large portion of our world today, much effort has been focused on making the representation of programs as small as possible. However I believe there is whole world to explore, even with regards to performance. If we would drop the notion that they should be as small as possible when read sequentually. In this piece I will try to give an overview of my work concering with Binary overlapping. Where the same byte sequence can be interperted as multiple programs if you start parsing with just one byte off.  



When an x86 CPU executes a program, a pointer points towards a seqeunce of bytes.
Which will get read byte for byte untill the CPU recognizes the instruction and it will execute it.
After execution is done, it will read the next byte until it recognizes it as an instruction repeating this process.
The CPU's today will not strictly adhere this model anymore and it might execute/parse instructions in advance under the hood already, but it still has to guarentee this model works. We will also consider this model because other software like disassemblers typically work this way.

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

If executed, it will first read the byte `0f`, this doesn't mean anything yet. So we will parse a second byte, `c0`. Still nothing, then we read `d0` and we get a specified instruction. In this case `NOP EAX, EAX`. Best described as "No OPeration executed with register EAX and EAX".

Afterwards we read two `NOP`s and a `XOR` insruction that `XOR`s both it's inputs. Hence zero'ing out all content.

What this program does is not really that interesting. Afterall it does nothing for a bit and then only zero's out an register. But after 3 years this is the most beautifull program I could think of.

First things first, why do we execute the same operation for the byte sequence `0f 0d c0` as for `90`. They share no bytes so what is up with this?

That is because if we look up `0f 0d c0` in the instruction manual, we can see that it is an instruction for prefechting hints. So it should not effect state on the cpu (or at least observable to us). And because of this the dissasmbler in which I generated this output decided it was equivelent to a NOP, and should be displayed as such.

However this above piece of code is only disassembled like that because I threw it inside a proper disassembler (XED), let's take a look at the output of objdump.

```
0f                      prefetch (bad)
0d c0 90 90 31          or     $0x319090c0,%eax
....
```

Three interstings things happend here, first of all it didn't decode it correctly. Which isn't to uncommon even in mainstream dissasemblers if you start looking for it[0]. 

The second interesting that happend we suddenly obtained a totally new instruction that we have not seen before by starting execution/parsing one byte later, while still using the same original byte sequence!

But most importantly of all, _what is the next executed instruction_?
We know it is should be the next byte in the stream, so looking at the first output: here we can see it should start with _FF_.

If we would continue this, we would have to completely indepedented programs inside the same original binary sequence!

# Usecases

This technique has 2 very interesting use cases. The original use case, and the reason why I came up with this idea, is obfuscation. 
Notice that in the second output, the output of the dissasembler objdump, can now theoritically be _completely different_ from what another actor, like your CPU, could read. Implying that maleware can be completely hidden from maleware analysis or automatic analysis tools without _any_ overhead. Only by exploiting a single parsing error. Which can be guaranteed  against most common dissasembler parsing techniques (this will the topic of another blog).

The second and probably more exciting use case is performance.
In the examples above we didn't really gave that much thought on what the programs are or are supposed to look like.

But let us consider a program P. If we take the second half of P, and we were magically able to alter the first half of P such that it's functionallity would be retained, and that if you would start parsing from the second byte it would execute the second half of the original P. Then we have effectively halfed our program size! And we could continue doing this  until we have reducecd it by a massive factor.


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


# Obtaining these binarires
We basically have 3 options that I'm aware of for obtaining these binaries.
We can write these binaries by hand, this is of course no option besides for very specialized cases and probably not worth it.

We could try alter the generation proccess of our compiler/assembler.
Computentionally generating binaries like this is really heavy and our current tooling are not really designed to support generating these binaries. 

## Generation 

Let me illustrate the complexity with the following example: 
Suppose we have 2 programs P and P', which we want to overlap. Meaning that the first byte of P' should be the second byte of P.
Consider that P is a simple program that wants to first execute an `add` followed by a `mov`, while P' wants to execute an `xor` followed by a `jmp`. For brevities sake I omitted possible arguments. 

Our magical super awesome assembler would look at P, recognizes that an we should output an binary encoding for `add`. Now it would look up which bytes are defined to represent `add` and it would emit it to some structure. 
Let's say for `add` the assembler emitted `10 AE`. 

Now before we continue, we have to satisfy that P' also get's generated correctly. Unfortantly for us the `xor` instruction is defined to be `06 84`.

In it's current form it seems as if `add` and `xor` are incompatable for binary overlapping.
Luckly for us, due to the incredibly complex nature there are a large set of transformations that can help to remedy this situation.

For instance it is not completely uncommon that an instruction has a different encoding. So while `add` was first defined to be `10 AE` at first, it might be the case that in the documentation an equivelent encoding exists for our usecase, so maybe we might substitude it with `30 06 84 20`. Enabling us to binary overlap at a cost of lengthening P compared to what we otherwise would.

So now we got `add` and `xor` to overlap. But now we run into the exact same problem for the second instruction for P, our generated encoding for `mov` may now only start with `20`!

This means that chance of being able satisfy the overlapping properties for instructions is __incredibily__ small. 
Hence we need every help we can get through combining many different techiques, such that we get an acceptable rate of being able to produce our desired binaries. 


### overview of generation techniques
Through the 3 years that I have been thinking about this problem, I accumlated a list of possible techniques:


* use a semantically equivelent instruction, like chaning `ADD 1` to `INC`remenent. Using `prefetchw` directives instead of `NOP`s.
* change an instruction encoding by using different prefixes or or encoding mechanisms. If an instruction ends with `48`, we can simply prepend `48` to any instruction and it will considered as a prefix. Guarenteeing overlap. 
* re-order instructions before they get used like done in out-of-order execution
* alter an instruction, but appending the inverse of the change. i.e `ADD 2` would become `ADD 5` with `SUB 3` appended. This technique can also be used with memory locations but requires much more setup 
* jumping ahead a few bytes S.T we can we can use later bytes instead of the current one in a given stream 
* inserting nop's or equivelants (or something like `add` and `sub` with the same argument). NOP's are great since these can have long arbitrary arguments see [1]
* let instructions use different registers 
* let instructions switch between using registers or memory

All of these techniques require extensive knowlegde of the program to a level typically not available to an assembler. 
Luckly if implemented gives an enourmous search space in which we can find potential candidate programs since we can keep infintily combining them.
In a future blog I will expand these entries.

## STOKE 

Implementing techniques so that arbitrary programs can be generated still requires a large engineering effort. However I temporarly circumvented this problem by modifiying STOKE[2] with a custom cost function for generating the largest program that still is overlapping. The result is a 70+ byte binary, albeit it did not presevere my original given sementics, it did proof the existance of these objects. Enough in my opinion to warrent further research into this direction.
Below is a picture of found binary, I also gave it the requirement of starting with an instruction that objdump wouldn't recognize. Meaning that it also demonstrates a method on how to hide programs from dissasemblers.

<insert image between objdump and xed> 









*take this definition as wide as you wish, it can be desired, undesired, valid or unvalid. You could techniqually have a desired unvalid program by catching UD exceptions. I just thought it would be helpfull to include some notion of correctness for sanities sake. That's also why I choose the wording sementacily valid. Since I sure as hell can't define what __that__ is supposed to mean




[0] Paleari, R., Martignoni, L., Fresi Roglia, G., & Bruschi, D. (2010, July). N-version disassembly: differential testing of x86 disassemblers. In Proceedings of the 19th international symposium on Software testing and analysis (pp. 265-274).



[1] Jämthagen, Christopher, Patrik Lantz, and Martin Hell. "A new instruction overlapping technique for anti-disassembly and obfuscation of x86 binaries." 2013 Workshop on Anti-malware Testing Research. IEEE, 2013.