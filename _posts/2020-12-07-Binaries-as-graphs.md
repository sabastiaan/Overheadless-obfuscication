---
layout: post
title:  "Binaries as a graph"
date:   2020-12-07 17:43:00 +0200
categories: x86 bin
---



Checking if a single x86 instruction text contains multiple valid execution paths is a hard problem to get your head around.
The hardest thing when it comes to this problem is the incredible lack of substructure based on an asinine encoding of an absurdly complex instruction set.
Although as much as I would like to hate (and love) on x86, this is a property of any variable width instruction set. 

Whenever we have a non-trivial problem like ours (if you know a solution within 5 minutes of I would love to hear this), a great step is to pick an arbtirary point of our problem, and see what it's properties are. This leads into insights of the conditions for which conditions must be met by your solution to be valid. 

So lets try this. I'm first gonna explore an example and then see what we can generalize it to.
Let us look at a single instruction Y in our program, surrounded by X and Z.
```
      X                     Y                       Z
+--------------+ +-------------------------+ +-------------------+
| 20 | 93 | 35 | | AE | 2F | D7 | E2  | 90 | | 6C | 0F | C0 | D0 |
+--------------+ +-------------------------+ +-------------------+
```

If this is overlapping, then somewhere some bytes with Y and Z form together an extra instruction.
For example it might be that from D7 onwards till 0F form an instruction like so:
```
          Y                       Z
+-------------------------+ +-------------------+
| AE | 2F | D7 | E2  | 90 | | 6C | 0F | C0 | D0 |
+-------------------------+ +-------------------+
             ^---------------------^
```

Or maybe it creates an even long instruction starting from 2F

```
          Y                       Z
+-------------------------+ +-------------------+
| AE | 2F | D7 | E2  | 90 | | 6C | 0F | C0 | D0 |
+-------------------------+ +-------------------+
        ^--------------------------------^
```

As you can see the result in the other stream could totally change, depending on where it allowed to start parsing.
This means that in what ever program Y gets put in, we cannot make assumptions on the overlapping program until we know the instruction before Y!

This makes us very, very sad. Since our problem became a lot harder. But do not despair.
Take the example of Y and Z again, the amount of places to start parsing from at Y are heavily restricted, one for each byte in Y, which is currently allowed to be at most 15 bytes. The ISA actually defines valid instructions that are longer than 15 bytes but those currently get rejected on every x86 CPU.


### testing all ways of overlap starting in Y (example)
What are all valid positions to start parsing and still be considered overlapping?

`Y[0]`: We know that parsing at `Y[0]` will sync again since these are literally the same instruction 

`Y[1]`: `2F .... C0` defined a valid instruction, which both used bytes of Y and Z and hence will be considered valid

`Y[2]`: `D7 E2` might define a valid instruction (assume it does for now), then we need to check if parsing at `90` is also valid

`Y[3]`: `E2` This might always be invalid, meaning that we can't extrapolate anything based from previous information

`Y[4]`: `90` In x86 this is famously a NOP instruction and is 1 byte, This means that the ending bytes of both streams now are the same and thus making it non-overlapping. Since this was also inside of the stream of execution would we have started at `Y[2]`, we know that starting at `Y[2]` would also make the binary non-overlapping.



We still have one more special case to consider for `Y`:
Note that here we considered all of the parsing streams starting in `Y`. However this does not cover all cases that `Y` is involved in!
Take the following example:

```
      X                     Y                       Z
+--------------+ +-------------------------+ +-------------------+
| 20 | 93 | 35 | | AE | 2F | D7 | E2  | 90 | | 6C | 0F | C0 | D0 |
+--------------+ +-------------------------+ +-------------------+
        ^--------------------------------------^
```

It might be that `Y` can be fully ___consumed___. But how this will change things will be part of my next blog. And we will assume that just doesn't happen for now.


#### What did we learn?
We now know that starting at a point in `Y`:

1. can lead to valid solutions
2. starting a byte later or earlier might invalidate the solution
3. and that a valid parsing stream does not mean we have a valid solution like in the last case.

There is one more thing subtle thing going on here, if we start at an byte, and an instruction is fully internal inside of another one, then it won't change the interface with other instuctions so to speak.



## Graph reduction
Our initial goal was to say if 2 streams overlap. We can make the visualization a bit simpler. 
If we lay out one stream sequentially, we know this will form a valid instruction stream.
It overlaps if the second path starting from the first instruction can be followed till the last instruction.
This looks like a graph problem!

If we make create a vertex for each instruction in our fixed stream, and an edge if they overlap, then we can easily solve the question :D
However this fails, since if the overlapping parsing stream between X and Y lands on `Y[2]`, but Y and Z only overlap if we start at `Y[4]`, then it would result in a valid graph, but the output won't be valid.

Thus we need make not  just 1 vertex for each instruction, but for each byte if we want to reduce our question to if there is a valid path!

This is how an reduction would look like for the above example: 

```
      X                     Y                       Z
+--------------+ +-------------------------+ +-------------------+
| 20 | 93 | 35 | | AE | 2F | D7 | E2  | 90 | | 6C | 0F | C0 | D0 |
+--------------+ +-------------------------+ +-------------------+
       +--+               +--+                      +--+ 
       |20|               |AE|                      |6C|  
       +--+               +--+                      +--+
                                                                                                                 
       +--+               +--+                      +--+ 
       |93| --------------|2F| -\     --------------|0F|  
       +--+               +--+   \   /              +--+
                                  -/-----------\                                         
       +--+               +--+    /             \   +--+ 
       |35|               |D7| ---               \--|C0|  
       +--+               +--+                      +--+
                                                                             
                          +--+  
                          |E2|   
                          +--+ 
                          
                          +--+  
                          |90|   
                          +--+ 
```





## What does this graph gives us? 
First of all I would consider this a major result. 
At least to my knowledge this is the first model of our problem which is easily understood and allows for ease of reasoning.
While this important considering that if one wants to transform a binary into something that exhibits the overlapping property, heavy modification will be expected (gut feeling).

Second of all, because it is now a graph, we can gain clear and _quantifiable_ metrics on our binary.
For instance even if a binary is not overlapping, but has a very dense graph, than the chance of us being able to chance it into something that we can modify will be much larger than if it  was sparse.

This will a guide for all of the changes that we can make as well as subsequent exploration.


### what to expect next
The fully consumed property does introduce extra complexity which needs to be dealt with. So this will be introduced in our next blog as well as a list of what graph structures can tell us about the binary.


#Key take away
We can reduce the question of "is a binary overlapping" into a graph. Giving us for the first time an easy mental picture of our problem and it's nuances. As well as guiding our future work





