---
author: Rui Wei
pubDatetime: 2026-01-25T17:43:00Z
modDatetime: 2026-01-25T17:43:00Z
title: Hello Everyone
slug: 
featured: false
draft: false
tags: []
description: An introductory blog post
---



Hi everyone, welcome to my first blog post.


## Code
```cpp
#include <iostream>
int main() {
	std::cout << "Hello world" << std::endl;
}
```




>[!proof] Modified Greedy Analysis
>**GOAL**: $\frac{C}{C*} \leq  \rho(n) \geq 1$
>$\text{Graph}\rightarrow \text{H, MST} \to \text{H', H + M (Min-cost Perfect Matching)} \to \text{R, Tour with Repetitions} \to \text{T, Tour remove Repetitions}$
> $cost(T) \leq cost(R)$ (triangle inequality) 
>$cost(R) \leq cost(H) + cost(M)$
>We know that $cost(H) \leq OPT$ since we just need to remove an edge from TSP 
>
>How do we find $cost(M)$ then?
>Let $T^*$ be the TSP tour on $G$ of cost $OPT$
>Given $S$ (subset of odd-degree vertices from $H$), we include these vertices in $T^*$ tour by adding shortcuts, let this be $T^*_S$
>=> $cost(T^*_S) \leq cost(T^*) = OPT$
>Let's include two perfect matchings $M_1$ and $M_2$ by alternating edges in $T^*_S$
>=> $cost(T^*_S) = cost(M_1) + cost(M_2)$
> $cost(M) = min\{cost(M_1), cost(M_2)\}$ - Since $M$ just needs to include all vertices from $S$ which is included in the tour $T^*_S$
>=>$cost(M) \leq \frac{cost(M_1) + cost(M_2)}{2} \leq \frac{cost(T^*_S)}{2} \leq \frac{OPT}{2}$
>$\therefore cost(R) \leq  cost(H) + cost(M) = OPT + \frac{OPT}{2} = \frac{3}{2}OPT$


> Hello world
> Naother 


Hello world
</br>


> [!info] Hello info
> Testing


> [!proof]  hello

> $\text{Graph}\rightarrow \text{H, MST} \to \text{H', H + M (Min-cost Perfect Matching)} \to \text{R, Tour with Repetitions} \to \text{T, Tour remove Repetitions}$

$$
\mathbf{P}(s) = \int_{0}^{\infty} e^{-st} \left[ \sum_{n=0}^{\infty} \frac{(\lambda t)^n e^{-\lambda t}}{n!} \cdot \mathcal{R}_n \right] dt + \oint_{\mathcal{C}} \frac{\nabla \times \mathbf{B}}{\mu_0 \varepsilon_0} \cdot d\mathbf{l}
$$

$$
\begin{aligned}
    \mathcal{L} &= \sqrt{\frac{1}{N} \sum_{i=1}^{N} (x_i - \mu)^2} \\
    \Delta \Phi &= \frac{\partial^2 \Psi}{\partial x^2} + \frac{\partial^2 \Psi}{\partial y^2} + \frac{\partial^2 \Psi}{\partial z^2}
\end{aligned}
$$

