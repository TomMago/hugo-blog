---
title: "Quantum Autoencoder"
date: 2022-07-12
tags: ["GSoC"]
author: "Tom Magorsch"
ShowToc: true
bibFile: content/posts/bib_lit.json
math: true
---

In my GSoC [project](https://summerofcode.withgoogle.com/programs/2022/projects/ePnjKlJs), I explore the use of Quantum Autoencoders for the analysis of LHC data. Autoencoders are an unsupervised learning technique, which learns a smaller latent representation of data. In this post, I want to show possible ideas for an Autoencoder circuit on a Quantum Device. 

# A naive Quantum Autoencoder

My first idea for a Quantum circuit closely follows the architecture of a classical autoencoder. The structure of the circuit is conceptually sketched in the following figure.
Here the Autoencoder has an input dimension of three and compresses the data to a single qbit.

![structure of a quantum autoencoder ciruit](../ae.svg#center)

Each line represents a single qbit and reading from left to right, consecutive transformations are applied to them.
The classical training data is first encoded into a quantum state indicated by $|\psi\big>$. This can be done with different encodings e.g. the angle encoding {{< cite "encodings" >}}.
After feeding the data into the first three qbits, an encoding block is applied. The encoder is a parameterized unitary transformation $U(\Theta)$. It consists of rotations on the block sphere and entanglement using CNOT gates. The parameters $\Theta$ are angles for the rotations which will eventually be learned when training the autoencoder.
After applying the encoder, a second parametrized circuit follows, which acts as the decoder.
The decoder only overlaps with the encoder at a subset of its qbits, in this example a single one. This qbit acts as the smaller latent space of the autoencoder. The rest of the decoder circuit acts on qbits which were initialized as $|0\big>$.
The goal of an Autoencoder is to learn to reconstruct the input data after compressing it to the latent space. To verify this, we can use a SWAP-test with three reference bits.
The reference bits were initialized to the input state of the data as well.
The SWAP-test can then measure the similarity of the input data state $|\psi\big>$ with the output of the decoder $|\phi\big>$.
This similarity $\big|\big<\psi|\phi\big>\big|^2$, called fidelity, can be measured at the readout bit.

Since we want to learn parameters that enable the autoencoder to reconstruct the input, we can use $$\mathcal{L} = 1-\big|\big<\psi|\phi\big>\big|^2$$ as loss function to train the parameters of the circuit.
The loss function can be used to optimize the parameters by [gradient descent](https://pennylane.ai/qml/demos/tutorial_backprop.html) similar to classical learning.

An example for a circuit of a $3\rightarrow 1 \rightarrow 3$ autoencoder is shown below. 

![quantum autoencoder circuits with gates](../QAE_circuit.svg#center)

In this example for simplicity the Encoder and Decoder consist of only a single layer using $R_y(\Theta_i)$ gates and entaglement by CNOT.
The SWAP-test is carried out using controlled SWAP-gates on the output of the decoder and the reference bits. The controlling bit is the last qbit which is used to readout the result of the SWAP-test. It is initialized as $|0\big>$ and prepared with a Hadamard gate, leads to the state $\frac{1}{\sqrt{2}}|0\big> + \frac{1}{\sqrt{2}}|1\big>$.
The controlled SWAP operation on two states $|\psi\big>$ and $|\phi\big>$ transfers the three qbit system into the state $\frac{1}{\sqrt{2}}|0,\psi,\phi\big> + \frac{1}{\sqrt{2}}|1,\phi,\psi\big>$. The $|\psi\big>$ could thereby be the state of an output qbit of the decoder and $|\phi\big>$ the state of a respective reference bit.
Applying another Hadamard gate to the SWAP-qbit transfers the state to $\frac{1}{2}\big(|0,\psi,\phi\big> + |1,\psi,\phi\big> + |0,\phi,\psi\big> - |1,\phi,\psi\big>\big)$.
When we now measure the first qbit, which is the auxillary SWAP-qbit, it will turn out to be $|0\big>$ with the probability
$$P(|0\big>)=\frac{1}{4}(\big<\psi,\phi| + \big<\phi,\psi|)(|\psi,\phi\big> + |\phi,\psi\big>) = \frac{1}{2} + \frac{1}{2}|\big<\phi|\psi\big>|^2.$$
The probability of measuring $|1\big>$ is therefore
$$P(|1\big>) = 1 - P(|0\big>) = \frac{1}{2}-\frac{1}{2}|\big<\phi|\psi\big>|^2.$$
To obtain the fidelity $|\big<\phi|\psi\big>|^2$ we measure the SWAP-qbit in the $Z$-basis.
Since the eigenvalues of $\sigma_z$ are $1$ and $-1$ the expectation value of the measurement calculates to
$$\big<q_8|\sigma_z|q_8\big> = 1\cdot P(|0\big>) + (-1)\cdot P(|1\big>) = |\big<\phi|\psi\big>|^2,$$ 
where $|q_8\big>$ denotes the SWAP-qbit. 
The output of measuring the last qbit of the shown circuit in the $Z$-basis therefore corresponds to the fidelity between the output of the decoder and the reference qbits.

# A simpler Quantum Autoencoder

The naive implementation discussed above can be simplified as shown in {{< cite "Ngairangbam2021" >}} based on {{< cite "Romero2016" >}}.
A $3\rightarrow 1 \rightarrow 3$ autoencoder circuit based on these ideas is displayed below. 

![simpler quantum autoencoder circuits with gates](../SQAE_circuit.svg#center)
 
The main idea is that, since the encoder circuit is a unitary transformation, if the fidelity of the non-latent qbits with some trash qbits which are just $|0\big>$ is one, the latent qbit will contain a compressed version $|\psi\big>_c$ of the input state $|\psi\big>$.
Therefore it is sufficient to maximize the fidelity between the non-latent qbits and some trash qbits to train the autoencoder.
Furthermore, a decoder is not necessary, since the decoder is just the adjoint of the encoder.
As we can see the main advantage of this method is that it needs way fewer qbits and works with a shallower circuit.
Nevertheless, it can still be used for compression, as the lower dimensional representation of the input data could be extracted from the qbit $(0,0)$. Moreover, the fidelity used for training can equally be used for anomaly tagging.

{{< bibliography cited >}}
