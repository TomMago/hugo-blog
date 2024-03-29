---
title: "Quantum Autoencoder"
date: 2022-07-12
tags: ["GSoC"]
author: "Tom Magorsch"
ShowToc: true
bibFile: content/posts/bib_lit.json
math: true
---

In my GSoC [project](https://summerofcode.withgoogle.com/programs/2022/projects/ePnjKlJs), I explore the use of Quantum Autoencoders for the analysis of LHC data. Autoencoders are an unsupervised learning technique, which learns a smaller latent representation of data. The quantum analog of a classical autoencoder equally aims to learn a smaller representation of data.

# A naive Quantum Autoencoder

My first idea for a Quantum circuit closely follows the architecture of a classical autoencoder. The structure of the circuit is conceptually sketched in the following figure.
Here the Autoencoder has an input dimension of three and compresses the data to a single qbit.

![structure of a quantum autoencoder ciruit](../ae2.png#center)

Each line represents a single qbit and reading from left to right, consecutive transformations are applied to them.
The classical training data is first encoded into a quantum state indicated by $|\psi\big>$. This can be done with different encodings e.g. the angle encoding {{< cite "encodings" >}}.
After feeding the data into the first three qbits, an encoding block is applied. The encoder is a parameterized unitary transformation $U(\Theta)$. It consists of rotations on the Bloch sphere and entanglement using CNOT gates. The parameters $\Theta$ are angles for the rotations which will eventually be learned when training the autoencoder.
After applying the encoder, a second parametrized circuit follows, which acts as the decoder.
The decoder only overlaps with the encoder at a subset of its qbits, in this example a single one. This qbit acts as the smaller latent space of the autoencoder. The rest of the decoder circuit acts on qbits that were initialized as $|0\big>$.
The goal of an Autoencoder is to learn to reconstruct the input data after compressing it to the latent space. To verify this, we can use a SWAP-test with three reference bits.
The reference bits were initialized to the input state of the data as well.
The SWAP-test can then measure the similarity of the input data state $|\psi\big>$ with the output of the decoder $|\phi\big>$.
This similarity $F$ is called fidelity
$$F(\ket{\psi},\ket{\phi})=\big|\big<\psi|\phi\big>\big|^2.$$
It can be measured at the readout bit.

Since we want to learn parameters that enable the autoencoder to reconstruct the input, we can then use the fidelity to construct the loss function $$\mathcal{L} = 1-\big|\big<\psi|\phi\big>\big|^2$$ to minimize.
The loss function can be used to optimize the parameters by [gradient descent](https://pennylane.ai/qml/demos/tutorial_backprop.html) similar to classical learning.

An example for a circuit of a $3\rightarrow 1 \rightarrow 3$ autoencoder is shown below. 

![quantum autoencoder circuits with gates](../QAE_circuit2.svg#center)

In this example for simplicity, the Encoder and Decoder consist of only a single layer using $R_y(\Theta_i)$ gates and entanglement by CNOT.
The SWAP-test is carried out using controlled SWAP-gates on the output of the decoder and the reference bits. The controlling bit is the last qbit which is used to read out the result of the SWAP-test. It is initialized as $|0\big>$ and prepared with a Hadamard gate, which leads to the state $\frac{1}{\sqrt{2}}|0\big> + \frac{1}{\sqrt{2}}|1\big>$.
The controlled SWAP operation on two states $|\psi\big>$ and $|\phi\big>$ transfers the three qbit system into the state $\frac{1}{\sqrt{2}}|0,\psi,\phi\big> + \frac{1}{\sqrt{2}}|1,\phi,\psi\big>$. The $|\psi\big>$ could thereby be the state of an output qbit of the decoder, e.g. $(0,2)$ and $|\phi\big>$ the state of the respective reference bit, e.g. $(0,5)$.
Applying another Hadamard gate to the SWAP-qbit transfers the state to $\frac{1}{2}\big(|0,\psi,\phi\big> + |1,\psi,\phi\big> + |0,\phi,\psi\big> - |1,\phi,\psi\big>\big)$.
When we now measure the first qbit, which is the auxiliary SWAP-qbit $(0,8)$ in the circuit shown above, it will turn out to be $|0\big>$ with the probability
$$P(|0\big>)=\frac{1}{4}(\big<\psi,\phi| + \big<\phi,\psi|)(|\psi,\phi\big> + |\phi,\psi\big>) = \frac{1}{2} + \frac{1}{2}|\big<\phi|\psi\big>|^2.$$
The probability of measuring $|1\big>$ is therefore
$$P(|1\big>) = 1 - P(|0\big>) = \frac{1}{2}-\frac{1}{2}|\big<\phi|\psi\big>|^2.$$
To obtain the fidelity $|\big<\phi|\psi\big>|^2$ we measure the SWAP-qbit in the $Z$-basis.
Since the eigenvalues of $\sigma_z$ are $1$ and $-1$ the expectation value of the measurement calculates to
$$\big<q_8|\sigma_z|q_8\big> = 1\cdot P(|0\big>) + (-1)\cdot P(|1\big>) = |\big<\phi|\psi\big>|^2,$$ 
where $|q_8\big>$ denotes the SWAP-qbit. 
The output of measuring the last qbit of the shown circuit in the $Z$-basis, therefore, corresponds to the fidelity between the output of the decoder and the reference qbits.

# A simpler Quantum Autoencoder

The naive implementation discussed above can be simplified as shown in {{< cite "Ngairangbam2021" >}} based on {{< cite "Romero2016" >}}.
A $3\rightarrow 1 \rightarrow 3$ autoencoder circuit without the SWAP test for measuring the fidelity of the output of the decoder is displayed below.

![simpler quantum autoencoder circuits with gates](../sqae2.png#center)

The Hilbert space $\mathcal{H}=\mathcal{H}_T\otimes\mathcal{H}_A\otimes\mathcal{H}_L$ can be written as the product of the subspaces for the so-called trash qubits $\mathcal{H}_T$, the data qubits which are not part of the latent space $\mathcal{H}_A$ and the compressed state at the latent bits $\mathcal{H}_L$. In this depiction, the trash bits are fed into the decoder by a SWAP operation ${V_T}_A$ between $\mathcal{H}_A$ and $\mathcal{H}_T$.
The input state to the autoencoder can be written as 
$$\ket{\Psi} = \ket{\psi}\_{LA} \otimes \ket{t}_T,$$
where $\ket{\psi}$ is the actual input data and $\ket{t}$ are the trash states which are initialized as $\ket{0}$.
The output state $\ket{\omicron}$ is the result of the unitary transformations and the SWAP applied to the input
$$\ket{\omicron} = U\_{LA}^\dagger V\_{AT} U\_{LA} \ket{\psi}\_{LA} \otimes \ket{t}_T.$$
The Fidelity between the input and the output $F(\ket{\Psi},\ket{\omicron})$ can be simplified
$$
\begin{align}
\begin{split}
F(\ket{\Psi},\ket{\omicron}) &= \big|\braket{\Psi|\omicron}\big|^2 \\\\
&= \big|\braket{\Psi|U^\dagger\_{LA}V\_{AT}U\_{LA}|\Psi}\big|^2 \\\\
&= F\big(U\_{LA}\ket{\psi}\_{LA} \otimes \ket{t}_T, V\_{AT} U\_{LA} \ket{\psi}\_{LA} \otimes \ket{t}_T\big)\\\\
&= F\big(U\_{LA} \ket{\psi}\_{LA} \otimes \ket{t}_T, U\_{LT} \ket{\psi}\_{LT} \otimes \ket{t}_A\big)\\\\
\end{split}
\end{align}
$$
where in the last line the SWAP $V\_{AT}$ exchanged $\mathcal{H}_A$ and $\mathcal{H}_T$.
We can now see that if 
$$U\_{LZ} \ket{\psi}\_{LZ} = \ket{\psi_c}_L\otimes\ket{t}_Z$$ 
the fidelity reduces to one.
Here $\ket{\psi_c}$ denotes a compressed version of the input state.
It is therefore sufficient to train the unitarity $U(\Theta)$ to separate the trash state $\ket{t}$. To do so, the decoder, which is the adjoint $U^\dagger$ is not needed. 
An example circuit is shown below. 

![simpler quantum autoencoder circuits with gates](../SQAE_circuit2.svg#center)

In this case, the SWAP test measures the fidelity between the trash qubits $\ket{t}$ and the output bits of the encoder which are not latent bits. 
As we can see the main advantage of this method is that it needs way fewer qbits and works with a shallower circuit.
Nevertheless, it can still be used for compression, as the lower dimensional representation of the input data could be extracted from the qbit $(0,0)$. Moreover, the fidelity used for training can equally be used for anomaly tagging.
Another useful advantage is the fact that we can upload more than one feature per qubit as shown in {{<cite"PerezSalinas2019" >}}, Because we don't have to compare the output of the decoder with the input data.

It should be noted, that the two fidelities for the first and the second approach are not identical. According to&nbsp;{{<cite "Romero2016" >}} the fidelity of the naive QAE is always less or equal to the simpler version. 

Currently, I am using this architecture and experimenting with different encoder circuits, as I am not aware of other fully quantum autoencoder architectures in the literature. It would be interesting to explore to what extent a potential quantum (variational ?) autoencoder could be used for generative purposes, which I will leave for the future.

{{< bibliography cited >}}
