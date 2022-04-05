---
layout: post
title: "Moduli spaces and the RS system, Part 1"
subtitle: "Introduction and the CM system"
---

First I should address the target audience for this post. Though this is the first post on any mathematical topic, the rest of this post is written assuming the reader is already well-versed in algebraic geometry and symplectic geometry. My hope is that at some point far down the line I'll have enough blog posts filling in all the gaps, but I want to describe what I actually *did* in this blog post so I have to start pretty far down the line. That being said, let us begin!

## Motivating questions

First I want to describe my perspective on my corner of math. I would say I fit squarely at the intersection of algebraic geometry, integrable systems, and mathematical physics.

More specificailly, my research focuses around finding algebro-geometric perspectives and interpretations on results in integrable systems. The mathematical physics comes in because I take a lot of insight and intuition from mathematical physics, and in particular supersymmetric gauge theory. In some sense I tend to view my mathematical goals as something like formalizing physics into mathematics: the physicists have a really good dictionary between integrable systems and geometry (Seiberg-Witten theory). It produces a lot of results about the geometry underlying certain integrable systems, but the calculations are ultimately done in physics and are not technically formal mathematics. My hope is to take the predictions from the physics calculations, and verify them using another way of relating algebraic geometry and integrable systems: the Hitchin system.

The goal of this series of posts is to exactly follow this blueprint through in a couple of extended examples. The first example is easier to describe, and essentially constitutes the contents of the paper <https://arxiv.org/abs/math/0603722> by my advisor Tom Nevins and David Ben-Zvi. The next example is exactly the contents of the paper <https://arxiv.org/abs/1909.08107> which was the main novel contribution of my thesis. There is a nice third example that would naturally fit into this blueprint, but actually constructing this example is the current focus of my research in this subject. At the end of this series I will hopefully include a few words about what I believe the shape that this last correspondence should take.

## The Calogero-Moser System

I believe the best place to begin this story starts at the Calogero-Moser system. This is an **integrable system** that describes the movement of many particles interacting in 1D subject to a very specific set of interacting forces. The system can be described in terms of the specific Hamiltonian function

$$
H_{CM} = \sum_{i=1}^N \frac{p_i^2}{2m} + g^2 \sum_{1 \leq i < j \leq N} \frac{1}{(q_i - q_j)^2}
$$

where $$q_i$$ and $$p_i$$ are the *positions* and *momenta* for particle $$i = 1, \ldots N$$. For our purposes, $$m$$ and $$g$$ won't play important roles. When it comes to the actual dynamics of the particles on the real line, the sign and reality of $$m$$ and $$g$$ play crucial roles in determining the long-term behavior of the system. Our focus is the geometry that underlies these systems, so we will actually be taking the positions and moment to be complex-valued. Meaning we can interpret the function $$H_{CM}$$ as a meromorphic function on $$\mathbb{C}^{2N}$$ with the first $$N$$ variables the $$q_i$$ and the last $$N$$ the momenta $$p_i$$.

Being symplectic geometers, we know we should actually understand this as the cotangent bundle $$\mathcal{M}_{CM,N} = T^* \mathbb C^N$$ which is symplectic for free. We may even consider replacing $$\mathbb C^N$$ by some space that avoids the diagonals $$q_i = q_j$$ for $$i \neq j$$, with the cotangent directions parametrizing the momenta $$p_i$$ which don't have the same restrictions.

As stated above, the Calogero-Moser system satisfies some property of being *integrable*. Because $$\mathcal{M}_{CM,N}$$ is symplectic, its ring of smooth functions (sheaf of regular functions) is a Poisson algebra (sheaf) with Poisson bracket $$\{f, g\}$$. The Calogero-Moser system being integrable means that there exist $$N$$ functions $$H_1 = H_{CM}, H_2, ... $$ on the $2N$-dimensional $$\mathcal M_{CM}$$ which have linearly independent covectors $$dH_i$$, and pairwise commute $$\{H_i, H_j\} = 0$$ for all $$i, j$$.

Just briefly, an integrable system is meant to model the situation that physicists often find themselves in where a physical system has enough conserved quantities to allow for the dynamics of the system to be solved by "quadratures" (integrals and solving algebraic equations). If this is the case, the functions $$H_i$$ define via the symplectic form a associated Hamiltonian vectorfields $$X_{H_i}$$, and the fact that the functions commute and are independent implies that the $$X_{H_i}$$ define an integrable distribution whose associated foliation has as its leaves the equi-value surface of all the $$H_i$$. In a future blog post I'll talk some more about the geometry of integrable systems.

## Next time

In the next post I will introduce the issue that we want to view the $$N$$ particles as indistinguishable, and so the above phase space needs to be modified. By the end we will have a nice geometric description of the Calogero Moser integrable system.
