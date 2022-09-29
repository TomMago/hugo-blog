---
title: "Data re-uploading"
date: 2022-09-15
tags: ["GSoC"]
author: "Tom Magorsch"
ShowToc: true
bibFile: content/posts/bib_lit.json
math: true
---

An important motivation for deep learning was the Universal Approximation Theorem which shows, that neural networks can theoretically approximate any function. When it comes to quantum machine learning, a similar statement can be made. Surprisingly a single qubit is sufficient, to perform the classification of arbitrary data distributions. 

# Universal Approximation Theorem 

The Universal Approximation Theorem (there are many versions with different constraints) states that the functions which can be expressed by a neural network with a single hidden layer and arbitrarily many units are dense in the space of continuous functions.
Considering a classification problem, we might have functions $f: I_m \to \Reals$, where $I_m = [0,1]^m$. The output of a neural network with a single hidden layer may be written as 
$$h(\vec{x}) = \sum^{N}\_{i=1}\alpha_i\sigma(\vec{w}_i\vec{x} + b_i), \tag{1}$$
where $\vec{w}_i$ and $b_i$ are the weights and biases of the hidden layer and $\alpha_i$ the weights of the output layer.
The function $\sigma$ is the non-linear activation function.
The function $h$ being dense in the continuous functions $f$ means, that for every $\epsilon$ we can choose the parameters in Eq. $(1)$ so that
$$|h(\vec{x}) - f(\vec{x})| < \epsilon \ \ \text{for all} \ \ \vec{x}.$$
This is a very powerful statement and enables neural networks to tackle very complex problems.

A proof for this theorem if $\sigma$ is a sigmoidal function, which means $\lim\_{x\to\infty}\sigma(x)=1$ and $\lim\_{x\to -\infty}\sigma(x)=0$, can be found in {{<cite "Cybenko1989" >}}.
The proof basically works by contradiction: If the space of all functions $h$ denoted by $S$ is not all of the continuous functions $f: I_m \to \Reals$ denoted by $C(I_m)$, then there is a linear functional $L: C(I_m) \to \Reals$ with $L(S)=0$ (or more accurately the closure of $S$).
The functional can be written as integral over a function $h$ with respect to some measure $\mu$. Since $L(S)=0$, which follows from the Hanh-Banach theorem, the integral over our neural network function $h$ with respect to the measure $\mu$ would vanish.
However one can show that for a sigmoidal function $\sigma$ integrals over terms of the form Eq. $(1)$ are non-zero for all non-zero measures, leading to the contradiction.

There are many variations of this theorem, especially {{<cite "Hornik1991" >}} provides a proof dropping the sigmoidal requirement.
This generalization applies to any nonconstant continuous activation function, which is bounded.

# Universal Quantum Circuit Approximation
In their paper {{<cite "PerezSalinas2019">}} the authors show that a similar proof can be done for the approximation capabilities of a PQC with a single qubit.
Let's consider some data $\vec{x}\in\Reals^n$ we want to classify. The data follows some classification function $f: \Reals^n \to O$ we want to approximate. In the simple case of binary classification, we might have $O=\\{0,1\\}$.
The idea proposed in {{<cite "PerezSalinas2019">}} is to subsequently apply parametrized gates with trainable weights and data uploads, effectively uploading the data many times. Hence the name data re-uploading. In their paper, the authors describe the motivation for the data re-uploading to be that classical neural networks effectively copy the input data when processing it. An example of a neural network with a single hidden layer is shown below. 

![Neural network with one hidden layer](../drc_3.png#center)

When e.g. passing the input data to the first hidden layer, the data is effectively passed to every unit of the hidden layer separately, thus "copied". In Quantum Mechanics, however, there is the No-Cloning-Theorem, which states that there is no unitarity $U$ that clones arbitrary input states.
This can be seen when assuming two states $\ket{\phi}$ and $\ket{\psi}$ which should be copied independently to a state $\ket{c}$.
The unitarity $U$ should therefore fulfill
$$
\begin{align*}
&\ket{S_1} = U(\ket{\phi,c}) = \ket{\phi,\phi},\\\\
&\ket{S_2} = U(\ket{\psi,c}) = \ket{\psi,\psi},
\end{align*}
$$ 
in order to universally clone the input states. Here we denote $\ket{\phi}\otimes\ket{c} = \ket{\phi,c}$.
The scalar product of the cloned states $\ket{S_1}$ and $\ket{S_2}$ can be written as
$$
\begin{align*}
\braket{S_1|S_2} =& \bra{\phi,c}U^\dagger U \ket{\psi,c} = \braket{\phi,c|\psi,c} = \braket{\phi,\phi|\psi,\psi} \\\\
=& \braket{\phi|\psi}\braket{\phi|\psi} = \braket{\phi|\psi}\braket{k|k}.
\end{align*}
$$
Since $\braket{k|k}=1$ we have
$$
\begin{align*}
\braket{\phi|\psi}^2 = \braket{\phi|\psi}, 
\end{align*}
$$
which is solved by $\braket{\phi|\psi}=1$ and $\braket{\phi|\psi}=0$. In the first case, the two states $\phi$ and $\psi$ are identical, which we don't want, since we want to clone different states with the same unitarity. In the second case, the two states are orthogonal.
The unitarity $U$ is therefore only able to clone orthogonal states. Non-orthogonal states can't be copied without some information loss.

To mimic the copying of input data to hidden notes, as it happens in classical neural networks, the authors, therefore, propose to upload the input data multiple times to a single qubit. An example of a DRC is sketched below.

![DRC example](../drc_2.png#center)

A single gate can be understood as a single perceptron, with the unitarity as activation function.

To investigate the capabilities of this circuit, we consider a general unitary transformation $U(\phi_1, \phi_2, \phi_3)\in\text{SU}(2)$. We can use this unitarity to upload data with $U(\vec{x})$ or apply transformations with trainable parameters $\vec{\phi}$.
In the case of data with only three features $\vec{x}\in\Reals^3$, we would construct the data re-uploading circuit (DRC) with depth $N$ as
$$\ket{m} = U(\vec{\phi}_N)U(\vec{x}) \cdots U(\vec{\phi}_1)U(\vec{x})\ket{0}.$$
After applying the gates, the qubit can be measured to access the state which arises from the PQC.
To reduce the number of gates and thus the depth of the circuit, we can incorporate the data upload and the parameters in a single gate.
A single processing unit as analogy to a single unit in a neural network is then written as 
$$L_i = U(\vec{b}_i + \vec{w}_i\odot\vec{x}),$$
where $w$ are the weights, $b$ the biases and $\odot$ denotes the elementwise Hadamard product. 
This already looks like the output of a single neuron!
The classifier then becomes
$$\ket{m} = L_N \cdots L_1\ket{0}.$$

Data with an arbitrary number of features can be treated by padding the data with zeros, so that the number of features is a multiple of three and then uploading the three-dimensional feature vectors $\vec{x}_j$ successively. In this case, a single processing unit $L_i$ is given as
$$L_i = U(\vec{b}^{(k)}_i + \vec{w}^{(k)}_i \odot \vec{x}^{(k)}) \cdots U(\vec{b}^{(1)}_i + \vec{w}^{(1)}_i \odot \vec{x}^{(1)}).$$

To see that this expression can approximate any function, we insert an explicit representation of $U(\vec{\phi})$ and summarize all transformations. For the general unitarity, we use
$$U(\vec{\phi}) = \mathrm{e}^{i\phi_2\sigma_z} \mathrm{e}^{i\phi_1\sigma_y} \mathrm{e}^{i\phi_3\sigma_z},$$
where we abbreviate $\vec{\phi} = \vec{b} + \vec{w}\odot\vec{x}$.
This is equal to 
$$U(\vec{\phi}) = \mathrm{e}^{i(w_1(\vec{\phi})\sigma_x + w_2(\vec{\phi})\sigma_y + w_3(\vec{\phi})\sigma_z)},$$
with
$$
\begin{align}
w_1(\vec{\phi}) = \frac{d}{\sqrt{1 - \cos^2d}} \sin\left(\frac{\phi_2 - \phi_3}{2}\right)\sin\left(\frac{\phi_1}{2}\right),\\\\
w_2(\vec{\phi}) = \frac{d}{\sqrt{1 - \cos^2d}} \cos\left(\frac{\phi_2 - \phi_3}{2}\right)\sin\left(\frac{\phi_1}{2}\right),\\\\
w_3(\vec{\phi}) = \frac{d}{\sqrt{1 - \cos^2d}} \sin\left(\frac{\phi_2 + \phi_3}{2}\right)\cos\left(\frac{\phi_1}{2}\right),
\end{align}
$$
where $d = \arccos\left(\cos\left(\frac{\phi_2 + \phi_3}{2}\right)\cos\left(\frac{\phi_1}{2}\right)\right)$.

A DRC with $N$ processing units can now be written as
$$\mathcal{U} := \prod^N_{j=1} \mathrm{e}^{i(w_1(\vec{\phi}\_j)\sigma_x + w_2(\vec{\phi}\_j)\sigma_y + w_3(\vec{\phi}\_j)\sigma_z)}.$$
Here the index of the $\phi$ denotes the index of the different weights and biases.
The product of Pauli-matrix exponentials can be simplified using the Baker-Campbell-Hausdorff formula
$$\mathcal{U} = \exp\left(i\sum^N_{j_1}\left[w_1(\vec{\phi}\_j)\sigma_x + w_2(\vec{\phi}\_j)\sigma_y + w_3(\vec{\phi}\_j)\sigma_z\right] + \mathcal{O}\_\text{corr}\right).$$

The correction term $\mathcal{O}\_\text{corr}$ proportional to commutators of Pauli-matrices.
The sum of the $w(\vec{\phi})$ terms can now be rewritten. Since all $w_i(\vec{\phi})$ are trigonometric functions, which are bounded to $[-1,1]$ and continuous, we can use the general version of the Universal Approximation Theorem and use the sum over $w_i(\vec{b}_i + \vec{w}_i \odot \vec{x})$ to approximate some continuous function $f_i(\vec{x})$ just like in the classical case
$$\sum^N\_{j=1}w_i(\vec{b}_j + \vec{w}_j \odot \vec{x}) = f_i(\vec{x}).$$

Since the correction terms are proportional to pauli matrices, {{<cite "PerezSalinas2019">}} argues, that they can be absorbed into the functions $f(\vec{x})$.

By optimizing the parameters with a classical optimization scheme, we can therefore approximate any function in terms of the final state (theoretically only with an infinite number of data re-uploads of course). To perform the optimization and classification, we can measure the qubit at the end of the circuit and compare the outcome with states which are predefined for the different classes. This way, it is possible to perform binary classification, but also multiclass problems can be treated by defining a label state for every class.

This approach can be extended to multi-qubit classifiers by entangling the different qubits.
I think, the data re-uploading approach definitely looks quite promising, since it proves the capabilities of QML.
In addition, it enables us to upload larger amounts of data on fewer qubits.
Furthermore, it seems to improve the robustness against noise see e.g. {{<cite "9415627">}}.
Currently, I am using DRCs in my autoencoders. In the future, I aim to explore different entangle schemes for multi-qubit DRCs. 


{{< bibliography cited >}}
