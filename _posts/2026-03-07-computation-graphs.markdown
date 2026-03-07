---
layout: post
title: "Computation Graphs: The Backbone of Modern ML"
date: 2026-03-07 11:14:29 -0500
categories: machine-learning fundamentals
---

Every time you train a neural network, there's a computation graph working behind the scenes. I spent a while not really thinking about what that meant, and once I did, a lot of things about PyTorch and TensorFlow clicked into place. So let's walk through it.

## What's a Computation Graph, Actually?

It's a directed graph that represents a math expression. You've got two kinds of nodes: **value nodes** (your variables, plus any intermediate results) and **operation nodes** (addition, multiplication, etc.). Edges connect them: values flow in, operations transform them, new values come out.

Here's a dead-simple example:

```
z = (x + y) * w
```

Drawn as a graph:

```
[x] ─┐
     ├─► [+] ─► [x+y] ─► [*] ─► [z]
[y] ─┘                     ▲
                            │
                           [w]
```

Notice that `[x]`, `[y]`, `[w]`, `[x+y]`, and `[z]` are all value nodes, while `[+]` and `[*]` are operation nodes. One thing that tripped me up early on: the values aren't just labels sitting on edges. They're proper nodes in the graph, which matters when we get to gradients.


## OK, but Why Should I Care?

Because **gradients**. When you're training a network, you need derivatives of your loss with respect to every parameter. That's a lot of chain rule.

The nice thing about having a graph is that it records every operation and every intermediate value along the way. So you can walk backwards through it, from output to inputs, applying the chain rule at each step. That's backpropagation, and the computation graph is what makes it possible to do mechanically instead of by hand.

Here's the part I find most satisfying: **gradients land on the value nodes**. After the backward pass, each value node holds the gradient of the output with respect to itself. So `x` doesn't just know it equals `2.0`. It also knows `dz/dx`.

PyTorch and TensorFlow handle all of this for you. They build the graph as your code runs, then walk it backwards when you call `.backward()`.

## Seeing It in Code

```python
import torch

x = torch.tensor(2.0, requires_grad=True)
y = torch.tensor(3.0, requires_grad=True)
w = torch.tensor(4.0, requires_grad=True)

z = (x + y) * w  # z = 20.0

z.backward()

print(x.grad)  # dz/dx = w = 4.0
print(y.grad)  # dz/dy = w = 4.0
print(w.grad)  # dz/dw = (x + y) = 5.0
```

Nothing special happened on the surface. You just wrote `z = (x + y) * w`. But PyTorch was quietly recording every operation, building up value nodes and operation nodes behind the scenes. When you called `z.backward()`, it walked that graph in reverse, dropping the computed gradients onto each value node.

## Building It From Scratch

Using PyTorch is nice, but it hides everything. Let's build a tiny autograd engine so we can see what's actually going on inside. The whole thing fits in about 40 lines:

```python
class Value:
    def __init__(self, data, children=(), op=''):
        self.data = data
        self.grad = 0.0
        self._backward = lambda: None
        self._children = set(children)
        self._op = op

    def __add__(self, other):
        out = Value(self.data + other.data, (self, other), '+')
        def _backward():
            # d(a+b)/da = 1, d(a+b)/db = 1
            self.grad += out.grad
            other.grad += out.grad
        out._backward = _backward
        return out

    def __mul__(self, other):
        out = Value(self.data * other.data, (self, other), '*')
        def _backward():
            # d(a*b)/da = b, d(a*b)/db = a
            self.grad += other.data * out.grad
            other.grad += self.data * out.grad
        out._backward = _backward
        return out

    def backward(self):
        # topological sort so we process nodes in the right order
        order, visited = [], set()
        def topo(node):
            if node not in visited:
                visited.add(node)
                for child in node._children:
                    topo(child)
                order.append(node)
        topo(self)

        self.grad = 1.0  # dz/dz = 1
        for node in reversed(order):
            node._backward()

    def __repr__(self):
        return f"Value(data={self.data}, grad={self.grad})"
```

Each `Value` is a node in the graph. When you do `a + b`, it creates a new `Value` wired to its inputs, and attaches a `_backward` function that knows how to push gradients back through that specific operation. That's the operation node and its local gradient rule, baked into one closure.

The `backward()` method does a topological sort of the graph (so we visit nodes in dependency order), seeds the output gradient at 1.0, then calls each node's `_backward` in reverse. Gradients accumulate via `+=` because a value used in multiple operations receives gradient contributions from each one.

Let's run the same example:

```python
x = Value(2.0)
y = Value(3.0)
w = Value(4.0)

z = (x + y) * w

z.backward()

print(x)  # Value(data=2.0, grad=4.0)
print(y)  # Value(data=3.0, grad=4.0)
print(w)  # Value(data=4.0, grad=5.0)
```

Same results as PyTorch, but now you can see exactly where those gradients came from. The `*` node pushed `out.grad * other.data` back to `self` (that's how `x` got `w`'s value of 4.0 as its gradient), and `out.grad * self.data` back to `other` (that's how `w` got `x + y = 5.0`).

This is basically what PyTorch does internally, just with GPU support, broadcasting, and a few hundred more operations. If you want to go deeper, this implementation is heavily inspired by Andrej Karpathy's [micrograd](https://github.com/karpathy/micrograd), a full autograd engine in ~100 lines of Python that supports enough ops to train small neural networks. Highly recommended.

## Static vs. Dynamic Graphs

This is one of those topics people used to argue about a lot more than they do now.

**Static graphs** mean you define the entire computation upfront, then execute it. TensorFlow 1.x worked this way. TensorFlow 2.x still does this under the hood when you use `tf.function`. The upside is the framework can see the whole picture and optimize aggressively. The downside is that debugging feels like pulling teeth. You can't just stick a `print()` in the middle of your graph.

**Dynamic graphs** get built on the fly as your code runs. PyTorch has always worked this way, and TensorFlow 2.x does too by default (eager mode). JAX falls somewhere in between (eager by default, static when you `jit`). The tradeoff is obvious: it feels like writing normal Python, at the cost of some optimization opportunities.

In practice, the industry settled this one. Everyone defaults to dynamic graphs now, and reaches for static compilation (`torch.compile`, `tf.function`) when they need the speed.

## Wrapping Up

I used to think of computation graphs as some abstract thing the framework dealt with. But once you see them for what they are, a record of every operation you ran, structured so you can walk it backwards for gradients, a lot of "why does the framework do it this way?" questions answer themselves.

Next time something weird happens during training, try thinking about the graph. It usually helps.