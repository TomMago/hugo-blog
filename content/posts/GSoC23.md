---
title: "GSoC 23 | Quantum Generative Adversarial Networks for HEP event generation the LHC"
date: 2023-10-12
tags: ["GSoC"]
author: "Tom Magorsch"
ShowToc: true
bibFile: content/posts/bib_lit.json
math: true
---

This is a summary of my 2023 GSoC [project](https://summerofcode.withgoogle.com/programs/2023/projects/ggoiGDQ5) with the [ML4SCI](https://ml4sci.org)-organization. In my project I designed and implemented a Quantum Generative Adversarial Network for the generation of HEP experiment data.
The full code for all my work can be found on [Github](https://github.com/ML4SCI/QMLHEP/tree/main/Quantum_GAN_for_HEP_Tom_Magorsch).
In the following post I will outline my work and describe some parts of the implementation and the results

### Event generation in HEP experiments

In high energy physics experiements like they are conducted at CERN, an integral part of the analysis process is the comparison of measurements with results expected based on predictions from our theory of nature, the Standard Model of particle physics.
![HEP experiment pipeline](../analysis_pipeline.png#center)
Generating these predictions is usually done by Monte Carlo simulations. Since these simulations are very demaning, there has been a vast amount of work on generative machine learning models e.g.&nbsp;{{<cite "Oliveira2017;Butter2019;Hariri2021" >}}. The main incentive is to speed up the simulation process by training a generative model like a GAN, from which one can then cheaply sample from.

### Classical GANs

Generative Adversarial Networks (GANs) are a class of unsupervised machine learning models proposed in&nbsp;{{<cite "Goodfellow2014" >}}. GANs aim to train a generator $G(z,\Theta_g)$ with a latent space $z$ and parameters $\Theta_g$ to replicate a reference probability distribution when sampling from the latent space $z$.

A GAN consists of two networks, the generator $G$ and a discriminator $D$. The networks are trained by playing a zero sum game, where the generator tries to generate samples which are as realistic as possible, while the discriminator tries to classify real data samples and tag samples generated by the generator as fake.

A more detailed description of classical GANs is given in [this post](https://www.tommago.com/posts/qgan/).

### A Quantum GAN for event generation

There have been several different proposals for Quantum GANs&nbsp;{{<cite "Lloyd2018;DallaireDemers2018;Niu2021" >}} including proposals for applications in HEP simulation&nbsp;{{<cite "Rehm2023;Rehm2023a" >}}.

In this project I focused on a Hybrid quantum classical model based on&nbsp;{{<cite "Zoufal2019" >}}. It consists of a quantum circuit acting as generator and a classical discriminator, which recives measured samples as input. Notably, this approach differs from many other approches in the literature, as it does not try to embed the continous data into quantum states through an embedding. We rather try to systematically discretize the data and develop a mapping to basis states of quantum system. The generator can then learn to prepare a quantum state whose distribution of basis states when measured resembles the discretized data. 

As a toy example, I take a dataset of detector images originating from Quark initiated Jets&nbsp;{{<cite "Andrews2019" >}} and stepwise scale this down to the size of $3\times 3$ with $M=2$ possible values per pixel. Below I show exemplary, how the average of all data samples scales down from a full $125\times 125$ detector image.

![data scaling](../qg_scaling.png#center)

As every possible state needs to be represented by a unique basis state, I represent every pixel with $\log_2M$ qubits, where $M$ is the number of discrete values a single pixel can take. The total qubits needed to represent an image of size $N\times N$ with $M$ discrete values per pixel is then $n = N^2\log_2M$. 

For our toy example we therefore have $n=9$ pixels. Of course this is a very simplified example, however it can work as a simple benchmark to test our model. 

Sampling from the generator we will obtain a measured basis stat which is a list of $0$s and $1$s representing the measured value for every qubit.

To map such a basis state $\ket{q_1q_2\dots}, q_i = 0,1$ to an image and vice versa, we proceed as following:
1. Reshape the list $[q_1,q_2,\dots]$ to $(N,N,\log_2 M)$
2. Convert the third dimension from an binary list to an integer
3. Divide by $2^M - 1$ to normalize

These images can then be passed into a classical deep neural network of your choice, meaning it is possible to utilize convolutional or graph neural networks as discriminator, as has been done in jet phyics before. Since the generator is trained on adversarially against the discriminator, we expect the generator to be able to generalize, given that the discriminator generalizes well on the data.

The training of the hybrid QGAN proceeds as following, for every epoch: 
1. We draw $N$ samples $s_i$ of basis states 
2. We evaluate the discriminator $D$ at theses samples giving $D(s_i)$
3. We get the probabilites for every sample beeing drawn by the generator $G$ as $p_i = G(s_i)$
3. We calculate the generator Loss $\mathcal{L}_G=\sum_i p_i \log D(s_i)$
4. We perform a step updating the generator parameters using $\mathcal{L}_G$
5. We evaluate the discriminator on a batch of real data samples $D(x_i)$
6. We calculate the discriminator loss as $\mathcal{L}_D=\frac{1}{N}\sum^N_i\log D(x_i) - \sum_i p_i \log D(s_i)$.
7. We perform a step updating the discriminator parameters using $\mathcal{L}_D$.

In the discriminator loss $\mathcal{L}_D$ the first term corresponds to the discriminator learning the real samples and the second term to learning to spot the fake samples generated by the generator.

I implemented this learning procedure using [pennylane](https://pennylane.ai/) and [pytorch](https://pytorch.org/). For that, I use a `qnode` with the pennylane pytorch interface

```
dev = qml.device("default.qubit.torch", wires=num_qubits)

@qml.qnode(dev, interface="torch", diff_method="backprop", cachesize=1000000)
def circuit(inputs, weights):
    for wire in range(num_qubits): qml.Hadamard(wires=wire)
    qml.StronglyEntanglingLayers(weights=weights, wires=list(range(num_qubits)))
    return qml.probs()
``` 

In this example, I return as measurement the probabilites of the basis states to use for the generator loss function. Note that I have to increase the `cachesize` for larger ciruits. As a unitarity in this example I use pennylane `StronglyEntanglingLayer`, though a more hardware efficient ansatz would be advantageous for execution on real hardware.

To perform the hybrid training with pytorch, we have to convert the `qnode` to a torch layer

```
weight_shapes = {"weights": (n_layers, num_qubits,3)}
qlayer = qml.qnn.TorchLayer(circuit, weight_shapes)
```

As discriminator we can build a feed forward neural network with the pixel number as input size.

```
class Discriminator(nn.Module):
    def __init__(self, input_size):
        super(Discriminator, self).__init__()

        self.linear_input = nn.Linear(input_size, 50)
        self.leaky_relu = nn.LeakyReLU(0.2)
        self.linear1 = nn.Linear(50, 20)
        self.linear2 = nn.Linear(20, 1)
        self.sigmoid = nn.Sigmoid()
        self.flatten = nn.Flatten()

    def forward(self, inputs):
        x = self.flatten(inputs)
        x = self.linear_input(x)
        x = self.leaky_relu(x)
        x = self.linear1(x)
        x = self.leaky_relu(x)
        x = self.linear2(x)
        x = self.sigmoid(x)
        return x

generator = qlayer
discriminator = Discriminator(N**2)
```

We can now optimize the parameters of these two models using pytorchs standard optimizers and the loss functions described above.

### Training results

The training results of the Qgan are shown below

![training progress of the qgan](../qgan_training.png#center)

We can see, that the KL divergence and the MSE between the average generated and data image decrease and converge in a controlled manner.
Therefore, the learned generator can generate the data distribution fairly well. It is worthwile to note that especially, there is no mode collapse here, as we are, in contrast to many other QGAN proposals, sampling from a quantum state, therefore mode collapse would mean learning a unitarity, which creates exactly a basis state. This is very unlikely and especially would not lead to a minimum in the loss function.

### Scaling and prospects

I think this is an interesting model for generative tasks on classical data. Of course we need more qubits than alternative models that embed continous data, however since the discretization scales with $\log_2 M$ in the long run this should not be the limiting factor.
I tried applying this model to larger images, however on a classical simulator the requried qubits were too many to simulate reasonably.

The main thing I would like to understand now is what advantages Quantum assisted machine learning can bring over classical methods. My main motivation for this model was the complexity theory based argument that by measurement quantum computers can efficiently sample from distributions, which are hard to sample for classical algorithms. However, I would like to understand this point better, develop an insight to what kind of data distribtuions this applies and especially make sure that the classical discriminator is able to learn these distribtuions.


# References

{{< bibliography cited >}}
