---
title: "Quantum Natural Gradient Descent"
date: 2022-08-27
tags: ["GSoC"]
author: "Tom Magorsch"
ShowToc: true
bibFile: content/posts/bib_lit.json
math: true
---

When training Variational Quantum Algorithms we aim to find a point in the parameter space that minimizes a particular cost function, just like in the case of classical deep learning.
Using the parameter-shift rule, we are able to compute the gradient of a Parametrized Quantum Circuit (PQC) and can therefore use that gradient descent method proven in classical machine learning.
However vanilla gradient descent can face difficulties in practical training which can be circumvented with Quantum Natural Gradient Descent (QNG).

# Natural Gradient Descent

When minimizing a cost function $\mathcal{L}(\Theta)$ the well known gradient descent iteratively updates the parameters $\Theta$ by descenting into the direction of the gradient
$$\Theta\_{t+1} := \Theta_t - \eta \nabla \mathcal{L}(\Theta)\big|\_{\Theta_t}.$$
Here and in the following all gradients are calculated with respect to $\Theta$
In Stochastic Gradient Descent (SGD) specifically the gradient $\nabla \mathcal{L}(\Theta)$ is approximated by the gradient of the cost funciton over a subset of the training data.
With this update rule, gradient descent implicitly assumes a euclidean geometry of the parameter space. This can be seen when writing the update rule as
$$\Theta\_{t+1} := \argmin_\Theta \left[\big<\Theta - \Theta_t, \nabla \mathcal{L}(\Theta)\big|\_{\Theta_t}\big> + \frac{1}{2\eta} \big|\big|\Theta-\Theta\_t\big|\big|^2_2 \right],$$
where a proximity term is added, just like in the lagrangian of a spring mass.
The equivalence to the gradient descent update rule can immediately be seen when solving the $\argmin$ by setting the derivative equal to zero. 

The choice of euclidean geometry does however not necessarily reflect the actual parameter space.
Since it gives equal weight to all parameters $\Theta_i$ ill conditioned situations can arise as e.g. shown below.

![Example of a two dimensional function](../grad.png#center)

The algorithm bounces over the valley and only slowly approaches the minimum. In the shown example the large step size aggrevates the problem. For SGD a careful tuning of the learning rate is therefore especially important. Optimizers like Adam can adress this problem by adjusting the step size based on previous gradients.
A reparameterization of the parameters space on the other hand could lead to problem way better suited for SGD.

So instead of using the euclidean metric $||\Theta||_2$ a distance measure for an infinitesimal vector $\text{d}\Theta$ on a curved manifold is given by
$$||\Theta||\_{g} = \sum\_{ij}g\_{ij}(\Theta)\text{d}\Theta_i\text{d}\Theta_j,$$
where $g\_{ij}$ is the Riemannian metric tensor.

For every physicist this seems very familiar. Of course the euclidean metric is the special case of $g\_{ij}=\delta\_{ij}$.
Using this general metric for the method of steepest descent S. Amari shows in {{<cite "Amari1998">}} that the gradient descent update rule becomes
$$\Theta\_{t+1} := \Theta_t - \eta G^{-1}\nabla\mathcal{L}(\Theta)\big|\_{\Theta\_{t+1}}\tag{1},$$
where $G^{-1}$ is the inverse of the metric $G = (g\_{ij})$.

The question remains how to determin the metric. In the framework of Information Geometry, instead of considering the parameter space, the optimization is performed on the so called statistical manifold.
A statistical manifold is a riemannian manifold, where every point corresponds to a probability function.

In our case, we may consider the manifold of likelihoods $p(x|\Theta)$ for the different possible parameters $\Theta$.
To measure the similarity between two probability distributions there exist different divergences, the most known one beeing the Kullback–Leibler (KL) divergence. For two distribution $p(x)$ and $q(x)$ it is defined as
$$D\_{KL}(p(x)||q(x)) = \sum_x p(x)\log\left(\frac{p(x)}{q(x)}\right)\tag{2}.$$
Note that formally the KL-divergence is not symmetric an thus is not a proper distance measure. However things work out for infinitesimal distance and thus it can be used to describe the manifold locally {{<cite "Martens2014">}}.

Let's try to rewrite our gradient update from SGD with the KL-divergence as instead of the euclidean metric:
$$\Theta\_{t+1} := \argmin_\Theta \left[\big<\Theta - \Theta_t, \nabla \mathcal{L}(\Theta)\big|\_{\Theta_t}\big> + \frac{1}{2\eta}D\_{KL}(q(x|\Theta)||q(x|\Theta_t)) \right]$$
To minimize this expression we set the derivative to zero
$$\nabla \mathcal{L}(\Theta)\bigg|\_{\Theta_t} + \frac{1}{\eta}\nabla D\_{KL}\left(q(x|\Theta)||q(x|\Theta_t)\right)\bigg|\_{\Theta\_{t+1}} = 0\tag{3}.$$
So to solve this we need the gradient of the KL-divergence, which we will approximate by taylor expanding the $D\_{KL}$ around $\Theta_t$. In the following we denote $D\_{KL}(\Theta||\Theta_t) := D\_{KL}(q(x|\Theta)||q(x|\Theta_t))$ for brevity. In second order we obtain
$$
\begin{align*}
D\_{KL}(\Theta||\Theta_t)\approx D\_{KL}(\Theta_t||\Theta_t) + \nabla D\_{KL}(\Theta||\Theta_t)\big|\_{\Theta_t}(\Theta-\Theta_t)\\\\ +\frac{1}{2}(\Theta - \Theta_t)^T H\_{D\_{KL}}\big|\_{\Theta_t} (\Theta - \Theta_t),
\end{align*}
$$
where $H\_{D\_{KL}}$ denotes the Hessian with respect to $\Theta$.
The first term obviously vanishes since the divergance for identical distributions is zero. The second becomes zero as well, which we can see if we insert the definition from Eq. $(2)$:
$$
\begin{align*}
\nabla D\_{KL}(\Theta||\Theta_t)\big|\_{\Theta_t}=&\sum_x \nabla p(\Theta)\big|\_{\Theta_t}\log\left(\frac{p(\Theta_t)}{p(\Theta_t)}\right) + p(\Theta)\nabla\log\left(\frac{p(\Theta)}{p(\Theta_t)}\right) \\\\
& =\sum_x \nabla p(\Theta) - p(\Theta) \nabla\log\left(p(\Theta_t)\right) = \nabla 1 = 0.
\end{align*}
$$
We can now insert the taylor expression for the KL-divergence in Eq. $(3)$ to obtain
$$\nabla\mathcal{L}(\Theta)\bigg|\_{\Theta_t}+\frac{1}{\eta}H\_{D\_{KL}}\bigg|\_{\Theta_t}(\Theta - \Theta_t)=0,$$
which leads to our update rule
$$\Theta\_{t+1} := \Theta_t - \eta H^{-1}\_{D\_{KL}}\bigg|\_{\Theta_t}\nabla\mathcal{L}(\Theta)\bigg|\_{\Theta_t} \tag{4}.$$
Comparing this with Eq. $(1)$ we can identify the metric $G$ with the hessian of the KL-divergence $H\_{D\_{KL}}$.
Rearranging the terms we can bring the hessian of the KL-divergence in the familiar form of the fisher information matrix
$${H\_{D\_{KL}}}\_{ij} = g\_{ij} = \sum_x p(x|\Theta)\frac{\partial \log p(x|\Theta)}{\partial \Theta_i}\frac{\partial \log p(x|\Theta)}{\partial \Theta_j}.$$
The fisher information matrix thus describes the local curvature of the statistical manifold.
With Eq. $(4)$ it constitutes the classical natural gradient descent.

# Quantum natural gradient descent

The optimization of PQCs is very similar to classical deep learning. We may have a quantum circuit with parameters $\Theta$. The resulting states of the circuit define a parametrized hilbert space $\mathcal{H}(\Theta)$.
We can define a distance measure $d$ between two states with an infinitesimal distance between the parameters
$$d\left(\ket{\psi(\Theta)}, \ket{\psi(\Theta + \text{d}\Theta)}\right) = \sum\_{ij} g\_{ij}(\Theta)\text{d}\Theta_i\text{d}\Theta_j,$$
where $g\_{ij}$ is the Fubini-Study metric {{<cite "Yamamoto2019">}} 
$$\text{Re}\left[\braket{\partial_i\psi|\partial_j\psi}-\braket{\partial_i\psi|\psi}\braket{\psi|\partial_j\psi}\right],$$
where $\ket{\partial_i\psi}=\partial\ket{\psi(\Theta)}\big/\partial\Theta_i$.
With this metric we can again write out and simplify our formulation of steepest descent to obtain an update rule for the quantum natural gradient descent proposed in {{<cite "Stokes2019">}}
$$\Theta\_{t+1} := \argmin_\Theta \left[\big<\Theta - \Theta_t, \nabla \mathcal{L}(\Theta)\big|\_{\Theta_t}\big> + \frac{1}{2\eta} \big|\big|\Theta-\Theta\_t\big|\big|^2\_{g(\Theta_t)} \right],$$
where using our metric $g(\Theta)$ we have the norm as the scalar product
$$\big|\big|\Theta-\Theta\_t\big|\big|^2_{g(\Theta_t)} = \braket{\Theta - \Theta_t, g(\Theta_t)(\Theta - \Theta_t)}.$$
Setting the derivative to zero like before directly leads to
$$\Theta\_{t+1} = \Theta_t - \eta g^+(\Theta_t)\nabla\mathcal{L}(\Theta)\big|\_{\Theta_t}.$$
Here $g^+$ denotes the pseudo-inverso of the metric tensor which is usually calculated as the [Moore-Penrose-Inverse](https://en.wikipedia.org/wiki/Moore–Penrose_inverse).

Computing the metric tensor can be very expensive, which is why {{<cite "Stokes2019">}} proposes to compute a diagonal or block-diagonal approximation of it. 

# Implementation 

Fortunately the computation of the diagonal and block diagonal approximations of the metric tensor are already implemented in [Pennylane](https://pennylane.ai)

I want to show a little example of the usage of QNG on a real dataset as I struggeled a bit with the implementation of the iteration over data. Suppose you have some `circuit` which takes the parameters and data as arguments. 
If you want to train the parameters you define some cost function e.g. a simple MSE

```
def cost(params, x, y):
    return (y - circuit(params, x)) ** 2
```

After initializing the parameters `params` we optimize them by iterating over the training data `x_train, y_train` and appling steps to the optimizer [`QNGOptimizer`](https://docs.pennylane.ai/en/stable/code/api/pennylane.QNGOptimizer.html) which is implemented in pennylane.
To compute the step however, we need the [metric tensor function](https://docs.pennylane.ai/en/stable/code/api/pennylane.metric_tensor.html), which is also implemented in pennylane.
As the metric tensor function can only be obtained for function with a single argument, namely the parameters to be trained, we need to define a lambda function for every data sample that only depends on the parameters. Same goes for the cost function.

```
opt = qml.QNGOptimizer(learning_rate)

for it in range(epochs):
    for j, sample in enumerate(x_train):        
        cost_fn = lambda p: cost_sample(p, sample, y[j])
        metric_fn = lambda p: qml.metric_tensor(circuit, approx="block-diag")(p, sample)
        params = opt.step(cost_fn, params, metric_tensor_fn=metric_fn)
        print(j, end="\r")

    loss = cost(params)
    
    print(f"Epoch: {it} | Loss: {loss} |")
```

Note that the data needs to be defined in pennylane with `requires_grad=False`.

The QNG can be quite useful in avoiding Barren Plateaus in training.
However of course computing the metric tensor takes time, which makes the QNG especially useful for models with a smaller number of parameters.

