---
layout: post
title:  "What is an ISA?"
date:   2020-06-13 23:09:46 +0200
categories: x86 bin
---



My journy started with a slighlty younger me with having the simple question "how many x86 instructions are there?". 
If we take C standerd library and we want to ask "How many functions does this library provide", we can probably get a decent answer pretty quickly. 
You can go to standard and count the defined functions there, take a look at it's source files, or maybe even some software tool just dumps it out for you with a simple command.

But to start answering this question I first needed to know what even an instruction is, where is defined, *how* is it defined? Can we easily find a list and what could we possibily do with it.

In this post we shall first explore what even an instruction is, how it is defined, and what does it do? 

Most desktops run on a intel or amd x86 CPU. These CPU's are large fat pieces of electronics that take a sequence of electronic input, and alter the state of it self, or something attached to it.
To make sense of the many things a CPU can execute we structed all the possible options it can perform into an *Instruction Set Architecture* or ISA for short. You can think of for example the instruction ```add``` that will take a number and increment some register somewhere in your computer.
So we talk about an x86 CPU, we mean that we are talking about a cpu that implements the x86 ISA.
You have a few flavors of ISA's, but we will ignore them because they are simply to different for what we are going to do.

Oddly enough there are 2 different ways to look at instructions, and you probably have seen them both in a limited fashion.
The first way is best given by example. ```ADD rax, 0x02 ```is an instruction, but so is ```MUL rbx, 0x40```, and ```MOV rax, rbx ``` too.
They all follow structure of ```name argument1, argument2```. Obviously there are slight variations on this like an instruction taking only 1 or no arguments, or special kind of arguments.
But they all follow this easy to comphrehand structure.

However how would a computer read this? Well they don't, they would read the binary encodings of this.
The creators of the ISA defined that when a CPU would read ```05 40``` that it would mean ```ADD``` 40 to the register called ```rax```.

Nearly everything your computer can read has an equivelant translation into the former form. The first form often also is called assembly and albeit rare, is still programmed in today to access special cpu features that is not exposed for example C. 
For example the C standard library doesn't state anything about virtualization so you need to resort to program in assembly or binary.

Internally all these instructions are translated into something called microcode, but how this actually works is kept secret (although you can still find some information). 



###### cut of point part 2?


Earlier I mentioned that there is a close correspondence between the assembly form and the binary form of the instruction set.
While close, it is important to note that this is not a one to one correspondence!

Take a look at the following picture: <insert picture of 3 add encoding's>
Here we can see that there are many different ways of the same instruction. 
The reason for this is the following:

Let's say that we want to be able to ```ADD``` a number to an arbitrary registers ```reg```
We need to specify that we want to ```ADD```, into what ```reg``` and what ```value```. Let's assume that each of these only requires 1 byte to encode in binary. 
As you can probably imagine this is an incredibly common instruction to execute. 
After a while we realized that we were mostly adding to ````rax```, hence we decided to resevere a special encoding to denote ```ADD RAX``` and ```value```, which only requires 2 bytes. 
Similary we can suspect that by far the most common value to be added is ```1````. Thus it might be benificial to again reserve some space to let a single byte encode ```ADD rax, 1```. 
Now we are suddenly in a position where 
So with this out of the way we can get into the real juice of all of this.


# Binary Overlapping 
Our programs don't consist of just one instruction but many of them stringed together. 
From now on I will call a program, i.e a possibily long list of instructions, ```P```. 
A different program will be called ```P'``` and ```P<sup>1</sup>``` will be the first *byte* of a ```P```.


As you could probably guess, a program runs *sequentually*. 
<<<show image of this
MOV RAX, 0 <-- first this 
ADD RAX, 4 
MUL RAX, 5 

MOV RAX, 0 
ADD RAX, 4 <-- then this 
MUL RAX, 5 


MOV RAX, 0 
ADD RAX, 4 
MUL RAX, 5 <-- lastely this 
>>>


But what if we want to do this procedure for ever? 
We obviously do not want to repeat all these lines, that would be messy, long and hard to understand.

Meet our new favorite instruction (and the last one you we have to know): JMP

With a jump we can change the next instruction to be executed, we can set conditions on when it should do that (for instance only if a specific register is 0 or only when negative).
Typically we can give a specific instruction a label like so:

START_LOOP:
0x00	MOV RAX, 0 
0x03	ADD RAX, 4
0x09	MUL RAX, 5 
0xAB	JMP START_LOOP

0xAC	MOV RAX, 0 

When JMP START_LOOP gets executed, it goes to the first MOV RAX, 0 and not the second one. Because it went to the label START_LOOP.
More precisly, it went to the *location* of START_LOOP in memory and there it found it's first instruction to be executed. MOV RAX, 0 in this case.
It is a *location* and not instruction! Why does this matter? 
Ask your self the following question, what would happen if we jump to START_LOOP + 1 instead of START_LOOP? 

Because we can manipulate the target of the jump instruction, an location, we can go to any part of the program. Which was made out of a long list of bytes.
So if we say JMP START_LOOP + 1, we jump to the second byte in the list of bytes starting at the position of START_LOOP. 
Recall that an instruction can be one byte. And in this case START_LOOP + 1 would instantly bring us to ADD RAX, 4.
However what if the first instruction was NOT 1 byte, but 3? We still go to the second byte, which is, well, we don't know what it is!

The proccesor doesn't care about this. For all it know's you intended to this for a good reason and happily executes from this location.
However we can rejoice since we DO know what is getting executed! If JMP START_LOOP executes MOV RAX, 0. It must have some list of bytes that it reads. And so we can just take a look at that list.
Let's pretend this is the following 3 bytes: 33 AA 00. If we start executing JMP START_LOOP + 1 we simply skip the first byte end up executing from the second byte: AA 00. 

But wait, I listed that it executed AA 00, however this doesn't have to be true. 
It will first only read AA and then will decide if it is an instruction to be executed or if it wants to read more bytes.
So it might only execute AA and then 00. Or it can execute AA 00 23 where 23 is the next byte after that.
For the first two options, we ended up in a situation where after AA and 00 were executed, and assuming they are not jumps, the second instruction would be executed as normal.
If we look at the third example however, suddenly the next instruction will be compromised of bytes that are part way through the second instruction. Again an instruction that was not listed in our original program! 
Continuing this will give us another program that is executed than what we originally wrote.

This "second" program exists entirely inside the representation of the first program, and they both share the same byte sequence. 
All the while the second program might execute something completely different. The only thing that was required was that we started in a different byte.
We can do the same trick of jumping ahead one byte on almost any program that exists. However most of them will be invalid an error would be given quickly. 
This makes everything a bit meaningless so we will also add the restriction that both programs must be *valid*.

Another thing that we want is that programs are actually different. 

As a by product of programs working sequently, if we ever start reading the same byte as the first byte of an instruction, every byte afterwards will be read as the same instruction. 
And we can't really call it different programs anymore. Therefore I would also like to propose a rule that the two programs may not have an instruction, which have the same starting byte (position wise inside the program) for an instruction.

Formally said, two programs P and P' are said to be overlapping if and only if 
Both are valid programs and 
The byte sequence of P' is included in P and 
No instruction in P shares the same starting byte as any instruction in P'
 

Finding overlapping programs is REALLY hard. A big factor is that most instructions can be represented quite are quite small in x86.