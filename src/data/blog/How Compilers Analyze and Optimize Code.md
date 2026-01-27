---
author: Rui Wei
pubDatetime: 2026-01-26T19:53:20+08:00
modDatetime: 2026-01-27T22:23:05+08:00
title: How Compilers Analyze and Optimize Code
slug:
featured: false
draft: false 
tags: [compiler]
description: How Compilers Analyze and Optimize Code
imageNameKey: how_compilers_analyze_and_optimize_code
csl: vancouver
---

## Table of contents

## Introduction
Sometimes we find ourselves setting the `-O2` or `-O3` flag when compiling  our code with `gcc` or `clang`  without actually  understanding what it does to our code, neglecting how these compilers actually work under the hood. However, setting these optimization flags at the cost of compile time may not always improve the performance of our code especially if our code is unpredictable. We first explore the need to optimize our code when generating the corresponding assembly code, followed by studying a framework that's used by compilers to harvest valuable insights to our program. Finally, we take a look at how compilers use these insights to properly optimize our code.

##  The Need for Optimization
In a trivial compiler implementation, we often treat the stack as if we have an unlimited amount of slots that we can simply place our variables in.  The compiler can simply look through the structure of the code to determine all possible variables that are declared within a function scope and simply reserve enough memory on the stack to slot these variables in on runtime.
```cpp
int f(int x) {
  int a = x + 2;
  int b = a * a;
  return b;
}
```
For example, the code excerpt above will only ever have 3 variables alive - `x, a, b` during the lifetime of the function call `f(int)`.  We can then simply tell our trivial compiler: *"Hey, reserve me 12 bytes worth of space to store these 3 variables when `f` gets called."*. The corresponding not optimized generated assembly code is shown below. Let's take a deeper look.
```asm
f(int):
    ; ...
    mov     DWORD PTR [rbp-20], edi ; move 'x' into its stack slot
    mov     eax, DWORD PTR [rbp-20] ; load 'x' from the stack into eax reg
    add     eax, 2                  ; a = x + 2
    mov     DWORD PTR [rbp-4], eax  ; move 'a' into its stack slot
    mov     eax, DWORD PTR [rbp-4]  ; move 'a' back into the eax reg
    imul    eax, eax                ; b = a * a
    mov     DWORD PTR [rbp-8], eax  ; move 'b' into its stack slot
    mov     eax, DWORD PTR [rbp-8]  ; move 'b' into the eax reg to return
    pop     rbp
    ret
```

![left](../../assets/excalidraw/How%20Compilers%20Analyze%20and%20Optimize%20Code%202026-01-26%2020.42.58.excalidraw.png)
The image on the left is a simplistic view of how the variables are assigned to their respective stack slot. Now you may have noticed that there are a lot of unnecessary `mov` instructions that transfer a variable value in between it's stack slot and into a register just for an arithmetic operation.  
</br>
This is where the bottleneck will incur, even in the best case where the stack slot is in the RAM, it takes about 4-5 CPU cycles to bring that data into a register, cycles that could have used for arithmetic operations instead.

You might have already noticed that it is possible to cut on these `mov` instructions between the stack and the register by simply using the registers only as shown below. The **goal** of this article aims to introduce a beginner's guide on how compilers identify these unnecessary instructions by allocating registers to produce efficient code.
```asm
f(int):
    ; ...
	mov     eax, edi
	add     eax, 2
	imul    eax, eax
	pop     rbp
	ret
```


## Dataflow Analysis Framework
Before we define the holy grail that is responsible for providing several useful information for our compiler to properly use to allocate registers. Let's take a look at an example of how this framework is used. [^1] 

### Live Variable Analysis
A variable, `v`, is *live* at a program point if, `v`, is defined before the program point and used after it. Simply put, we are interested in where our variables are defined and where it's being used at. This analysis provides us valuable information as we can safely tell  our compiler that if two variables are not live at the same time then they can share the same register throughout the function call, reducing the need for a stack slot.

The intuition and the algorithm behind the framework is not trivial and may require extensive reading on your own free time.  While I try my best to explain,  feel free to skip this part as you would still be able to understand how compilers allocate registers just by assuming the information provided by the framework is given.

Let's set some terminology.
- $use[s]$: set of variables used by $s$
- $def[s]$: set of variables defined by $s$
- $in[s]$: set of variables live on entry to $s$
- $out[s]$: set of variables live on exit from $s$

For example, `a  = b + c` implies that $use[s]  = \{b, c\}$ and  $def[s] = \{a\}$. Intuitively, $use[s] \subseteq in[n]$.  Now let's take the code excerpt defined above and transform it into a graph.

![../../assets/excalidraw/How Compilers Analyze and Optimize Code 2026-01-26 21.35.10.excalidraw](../../assets/excalidraw/How%20Compilers%20Analyze%20and%20Optimize%20Code%202026-01-26%2021.35.10.excalidraw.png)

How would you come out with an algorithm that would basically fill out the necessary information as shown in the graph above?

> [!proof] Backtracking Algorithm
>  1. For each variable, $v$ (each statement of code - represented as a node)
>  2.  Look at each node that use $v$ and try all paths tracing backwards through the graph until either $v$ is defined or we have visited a previously visited node
>  3. For all those nodes within the path, mark $v$ as live on entry into that node

While the algorithm is simple, we are traversing the same path  many times across different variables, let's try to think of a better algorithm given these additional properties that we can identify.
1.  $use[n] \subseteq in[n]$: "A variable must be live on entry to this node if we are using it"
2.  $out[n] - def[n] \subseteq in[n]$: "If a variable is live on exit of this node and we didn't define it, means it must have been live on entry"
3.  $in[n'] \subseteq out[n], if\ n' \in succ[n]$: "If a variable is live on entry to a successor node of $n$, it must be live on exit from $n$"

Upon analyzing these properties, we can conclude that the $in[n]$ of a node depends on what our current statement uses. Also, whether or not variables are in $out[n]$ depends if my successor statements use those variables.

To kick start the algorithm,  we start of with a rough guess by treating each statement independently and initializing $in[n] = out[n] = \{ \}$. This enables the most optimization opportunity for each node since there are no live variables at the same time and hence we can allocate any register to the variables that are used in this node, which is ultimately the best case (do note that the starting case is not necessarily true/valid). The goal of this algorithm is to iteratively converge till all nodes satisfy the three properties. You may notice that as the algorithm goes on, the set of $in[n]$ and $out[n]$are monotonically increasing - you can think in this way:
- For example, the statement, $n$,  `int a = b + c` , this node uses `b` and `c`, therefore `in[n]` would expand based on property one, also if successive statements, $n'$, use other variables that this statement didn't define, such as `x, y, z`, this implies that those variables must have been defined before this node and therefore are in their `in[n']`. Since the liveliness of those variables must have been carried out through the statement, $n$, `out[n]` must include those variables `x, y, z` as per the third property.

We summarize the **update pattern** that occur throughout the algorithm as such.
- *Control-flow constraint*: $out[n] := \cup_{n' \in succ[n]}in[n']$
- *flow function*: $in[n] := use[n] \cup (out[n] - def[n])$

The algorithm terminates when all $in[n]$ and $out[n]$ have no changes. The algorithm is also guaranteed to terminate because there will always be a finite number of variables. This algorithm is a **fixed-point computation**, closely related to concepts from numerical analysis, which are completely out of scope. We summarize the algorithm's pseudocode below.

> [!algo] Live Variable Analysis pseudocode
> * for all n, $in[n] := \emptyset, out[n] := \emptyset$ 
> * repeat until no change in $in[n], out[n]$
> 	* for all n
> 		* $out[n] := \cup_{n' \in succ[n]}in[n']$ // "update our out based on what successors use"
> 		 * $in[n] := use[n] \cup (out[n] - def[n])$ 
> 	* end
> * end	


### General Dataflow Analysis Framework
In the live variable analysis, we associated every program point, a node,  a *data-flow value*  that contains an element of the domain of that analysis. For example, the domain of the liveliness analysis would be the set of all possible variable combination, e.g. $\{\{\}, \{a, b\}, \{a, b, c\}, \dots \}$.

In any form of analysis such as *reaching definitions* and *available expressions*, each statement, a node, holds  a value of the domain at any point in time of the algorithm. Throughout the algorithm, each node follows their respective **update pattern** and propagates new information to other nodes. Every analysis propagates information either forward or backward depending on the nature of the analysis. Where $\sqcap$ can be either $\cup, \cap$ depending on the nature of the analysis.
- *forward*:
	-  $in[n] = \sqcap_{n' \in pred[n]}out[n']$
	- $out[n] = F_n(in[n])$
- *backward*:
	-  $out[n] := \sqcap_{n' \in succ[n]}in[n']$
	- $in[n] = F_n(out[n])$

For example, in the liveliness analysis that we mentioned above, a variable is in $out[n]$ only if the successive statements use those variables and are in their $in[n']$, and $\sqcap$ is defined to be $\cup$ since we are interested in all variables used by successors. Therefore, live variable analysis follows a backward propagation. We now define, $F_n$ as the flow function that is basically responsible for how information is propagated across the node: what is added and what is removed. The general iterative (forward) analysis is shown below:

> [!algo] Framework pseudocode
> * for all n, $in[n] := T, out[n] := T$        // $T$: "maximal information (highest optimization opportunity)"
> * repeat until no change in $in[n], out[n]$
> 	* for all n
> 		* $in[n] := \sqcap_{n' \in pred[n]}out[n']$ 
> 		 * $out[n] := F_{n}(in[n])$
> 	* end
> * end	

In summary, to conduct a dataflow analysis,  the implementer needs to figure out three important properties and he can simply just plug them in the algorithm to obtain useful information for the compiler to conduct optimizations.
1. Direction of the propagation
2. $T$ - the maximal set for the initialization - which also hints if $\sqcap$ should be $\cup$ or $\cap$ too which determines the control-flow constraint
3. $F_n$ - the flow function or also known as the transfer function

To understand deeper about the framework, the reader should read more about hasse diagrams and partial ordering. We won't be going too deep into that.

### Example
Now let's take a look at an example of a forward propagation: *reachable definition analysis* and learn how to figure out the properties required for us to use the framework. This analysis provides us information on whether or not we can propagate constants or remove dead code. A definition, $d$, of variable, $x$, reaches a point $p$ if there is a path from $d$ to $p$ where there is no re-definition of $x$. In this case, each statement, a node, would have a  unique ID and the domain for this analysis would be the set of all possible combinations of node ID. Let's take a look at the graph for the code snippet below.

```cpp
void f(bool c) {
	int x = 0;
	if (c) {
		x = 1;
	} else {
		x = 2;
	}
	int y = x;
}
```
![](../../assets/excalidraw/How%20Compilers%20Analyze%20and%20Optimize%20Code%202026-01-27%2018.36.25.excalidraw.png)

So from the image above, since node 2 and 3 redefines `x`, `out[2]` and `out[3]` does not include node 1.

Before we define the **update pattern**  for this particular analysis (recall that we have to define both the flow function and the control-flow constraint), it might be helpful to figure out both the direction of the propagation and $T$ (the initialization value that contains the "maximal optimization opportunity") first.  In this case,  what should the size of `in[n]` be such that we can provide the best optimization? From the graph above, $in[4] = \{2, 3\}$, which clearly shows that there are two definitions of $x$ coming from node 4 predecessors, $out[2]$ and $out[3]$. Because of the double definition, we are unable to optimize by propagating either of the constant value in node 2 or 3 (`x = 1` and `x = 2`). Therefore, having less information in `in[n]` provides the most opportunity for optimization and therefore our  $T$/"top" is initialized as the empty set for this analysis.

Now, how do we determine the direction of the propagation?  For a node to know that it contains reachable definitions of variables, it needs to receive update from its predecessors, therefore it must be a forward propagation. Now let's define the update pattern.

Now to define the control-flow constraint, we need to determine the "meet" operator, $\sqcap$, in this case it will be $\cup$ since our $T$ is an empty set. Now for the flow function, $F_n$. Let's take an example $u = a + 1$. We define
- $gen[n]$: the set that contains the statement, $n$, that "generated" a definition of variable $u$
- $kill[n]$:  the set that contains the node ID of previous statements that created a definition of variable $u$

In this case, $gen[n]$ is "similar" to that of $use[n]$ while $kill[n]$ is "similar" to that of $def[n]$ in the live variable analysis. Now we can define our flow function, $f_s(in[n]) = out[n] =  gen[n] \cup (in[n] - kill[n])$, which basically means - *"on exit of this statement, statements that still have reachable definition includes my own definition for $u$ plus other definitions of other variables excluding the previous definitions of variable $u$"*. Simply put, "because this statement defined $u$, all past definitions  of $u$ are rendered invalid from now  on".

Now with all the three properties known, I simply need to plug these information into the framework and voila, we have useful information that our compiler can use to optimize our code. 

> [!algo] Reaching Definition pseudocode
> * for all n, $in[n] := \emptyset, out[n] := \emptyset$        // $T$: "maximal information (highest optimization opportunity)"
> * repeat until no change in $in[n], out[n]$
> 	* for all n
> 		* $in[n] := \cup_{n' \in pred[n]}out[n']$ 
> 		 * $out[n] := gen[n] \cup (in[n] - kill[n])$
> 	* end
> * end	

## Register Allocation
So now we know how compilers conduct dataflow analysis under the hood to obtain valuable information. How do compilers use those information to optimize our code? Let's take a look at how compilers make use of the information provided during the live variable analysis to properly allocate registers to variables.

Note that the live variable analysis provides all possible live variables at any program point. That would mean, none of these live variables at this program point can share registers. Let's treat each variable as a node and an edge exists between two nodes if both variables are live at the same time at any point in time. Let's treat each register as a color too. Does this problem sound familiar? It's exactly a [graph coloring](https://en.wikipedia.org/wiki/Graph_coloring) problem. But the problem is that we can't efficiently find a k-coloring of the graph since it's a [NP-complete ](https://en.wikipedia.org/wiki/NP-completeness) problem. So how do we assign registers to colors? Let's take a look at Kempe's Algorithm.

> [!algo] Kempe's Algorithm (Recursive)
> * Base Case: A single node just assign it a random color among the k-color
> * Recursion:
> 	* 1. Simplify the graph by finding a node with degree $< k$ and cut it out of the graph (remove the node and edges)
> 	* 2. Recursively k-color the remaining sub-graph
> 	* 3. On the way up, there must be one free color available for the "deleted" node since its degree was $< k$. Pick that color and color it.

What makes Kempe's Algorithm special is that we don't exactly need to determine the exact value of $k$ to color our graph. In this context, because we have limited number of registers, we can simply fix the value of $k$ to that of the number of registers. 

![](../../assets/excalidraw/How%20Compilers%20Analyze%20and%20Optimize%20Code%202026-01-27%2020.41.33.excalidraw.png)


Now consider the case above, suppose we are trying to 2-color this particular graph, after removing the first node, we do not have a node with degree $< k$, in this case Kempe's algorithm wouldn't work anymore. However, upon closer inspection, if the graph have no nodes with degree $<k$ that simply means there exists a sub-graph such that there are $k$ variables live at the same time. Intuitively, we can simply spill the additional live variable into the stack (each address of the stack slot is a unique color), now can modify Kempe's Algorithm: 
- While there is no node with degree $<k$, keep spilling the node into the stack by assigning it a "different" color

This forms the basis of how compilers assign registers to variables to minimize the number of stack slots used so as to minimize the inefficient usage of `mov` instructions. There are some more interesting ways we can tweak our algorithm to improve the coloring process, I list and explain some of them.

- *Optimistic Coloring*:
	- Remember how we spill a variable out into the stack when there is no sub-graph of degree $<k$? Even though the node is marked for spilling, it is very much possible to color that node as the algorithm converges back up. Try to think of a small graph example.
- *Pre-colored Nodes*:
	-  On X86 the `IMul` instruction requires that the result goes into $eax/rax$ register. We can simply pre-color those nodes.
- *Choosing good colors*:
	- Sometimes two variables that are live at the same time could simply just mean $x = b$ and we don't exactly need two registers for $x$ and $b$ especially when $b$ is dead afterwards.

## Conclusion
We first discussed what was live variable analysis and went on to figure out how we were able to determine the set of live variables existing at every program point. We then generalized the framework we used to analyse our program and extract useful information at each step of the program point.  Afterwards, we took this framework and tried to gather some intuition on how  to apply the framework to conduct the reaching definition analysis. Now we have figured out how compilers gather insights of our program, we explored how live variable analysis was used by the compiler to figure out how to best allocate registers to our local variables so as to minimize the number of inefficient `mov` instructions to optimize our assembly code.

---

[^1]: Aho, Lam, Sethi, Ullman. *Compilers: Principles, Techniques, and Tools*, 2nd Edition 