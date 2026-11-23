# Ising model and bootstrap percolation

## 1. Ising model

The **Ising model** is a mathematical model of interacting binary variables used to study how local interactions and randomness produce collective order, disorder, and phase transitions.

- Originally developed to study magnetism.
- Used to understand phase transitions, collective behavior, and optimization.
- **Key idea:** Simple local interactions can produce large-scale order.

## 2. Bootstrap percolation

A mathematical model of **how activation spreads through a network or grid** according to a threshold rule.

- Initially, some nodes are active.
- An inactive node becomes active when at least $k$ of its neighbors are active.
- Once activated, a node stays active permanently.
- **Key idea:** Local activation rules can cause a cascade that eventually activates the entire system.

## 3. Why is it called bootstrap percolation?

The name combines two ideas:

**Bootstrap:** Refers to a self-reinforcing process. Previously activated nodes help activate additional nodes, which then activate even more nodes.

**Percolation:** Refers to how something spreads through a medium or network, analogous to water filtering through porous material.

The distinction from ordinary percolation is important:

- **Ordinary percolation:** Studies connectivity through randomly occupied sites or edges.
- **Bootstrap percolation:** Studies how an initially active set grows through repeated threshold-based activation.

**In short:** Bootstrap percolation describes a network that can progressively activate itself through local interactions, potentially producing a system-wide cascade.

## Related concepts

See [Diffusion models](../probability/diffusion_models.md) for how randomness and learned denoising generate structured data, and how this connects to statistical mechanics.
