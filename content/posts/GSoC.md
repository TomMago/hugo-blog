---
title: "GSoC 22 | Quantum Autoencoders for HEP Analysis at the LHC"
date: 2022-09-21
tags: ["GSoC"]
author: "Tom Magorsch"
ShowToc: true
bibFile: content/posts/bib_lit.json
math: true
---

This is a summary of my 2022 GSoC project with [ML4SCI](https://ml4sci.org).
The ML4SCI organization accustoms different projects of machine learning applied to scientific problems, many connected to high energy physics.
Big thank you to Sergei Gleyzer for the supervision and support.

# Abstract
The Standard Model of particle physics is a theory that describes the fundamental particles and the interactions between them.
While it has extensively been tested and was able to correctly predict experiments to an impressive degree, there are [multiple reasons](https://en.wikipedia.org/wiki/Standard_Model#Challenges) to believe that it cannot be a complete description of nature.
In the search for physics beyond the Standard Model (BSM), even though the large hadron collider (LHC) produced a large amount of data, no conclusive evidence of new physics could be found yet. A promoising method to uncover new physics in the large amount of data is the use of anomaly detection techniques, which can be used to tag anomalous events. A well known method of deep anomaly detection is the use of autoencoders, which have been applied to the task of anomaly tagging in HEP data before. In my project study the use of quantum machine learning models to the task of anomaly tagging, to investigate if quantum computers can enhance these analyses.

# The project
In order to apply to GSoC with ML4SCI you have to complete some preliminary screening tasks, and write a proposal for your project.
Solving the tasks took some time, but they were very interesting and already prepared well for the upcoming project. You can see my proposal [here](../QAE_propsal.pdf), if you are interested. Of course the project deviated a bit from the initial plan, but all in all I followed the plan outlined in the proposal.

## Anomaly tagging
With the large amounts of data produced by the LHC and potentially produced in the future by the [HL-LHC](https://en.wikipedia.org/wiki/High_Luminosity_Large_Hadron_Collider), new analysis techniques pose interesting tools to detect new physics. I think the BSM physics is certainly hiding somewhere in the data, uncovering it is just a question of finding the needle in a haystack, due to its elusive nature. I especially like the idea of unsupervised techniques, since it is a way to conduct a model independent search. Since there are many different BSM models I think model independent searches make a lot of sense. There has previously been a good amount of reasearch on the application of unsupervised methods to new physics searches, e.g. </br>{{<cite "Kasieczka2021;Fraser2021">}}. 
Many studies apply anomaly detectio to detector images. Since Autoencoders are one of the most prominent deep anomaly detection models, they have been applied to these anomaly studies as well. I specifically want to highlight {{<cite "Finke2021">}}. In their study, the authors use a convolutional autoencoders to tag top quark initiated jets as anomalous, in samples of QCD initiated jets. In particluar they highlight, that there is a complexity bias between the QCD and top samples. This refers to the fact, that a convolutional Autoencoder trained on top jets can not tag QCD jets, since the top jets have a higher intrinsic dimensionality, which enables the Autoencoder to work on the "easier to reconstruct" QCD samples. I think this can defenitely be a problem as it sets constraints on the type of new physics we are able to uncover and thus its an interesting matter to investigate when working on this type of anomaly detection.

## Datasets

In the project I used different datasets, e.g. MNIST for validating ideas and code samples or ECAL images of electrons and photons.
However, my main focus was on a dataset of detector images of quark and gluon initiated jets, which is described in </br>{{<cite "Andrews2019">}}. The dataset only contains jets of light quarks ($u$, $d$, $s$). The original dataset contrains 125x125 images of the electromagnetic calorimeter, hadronic calorimeter and the tracks for every event respectively. Below I show the average images of the three channels for quarks and gluons respectively. Note that I normalized the images by dividing them by the value on the largest pixel respectively. Now I'm not completely sure if this is the best way to do it, but since I need the pixel values to be properly distributed in $[0,1]$ this was the most obvious way for me.
The HCAL channel was generated at a lower resolution and upscaled to 125x125 resulting in a corser pixeling than the other channels.

![Averages of jets](../jet_samples.png#center)

We can see, that the Gluon initiated jets show a slightly wider cone, however the differences are quite small, which makes the differentiation of these jets a very hard taks. It becomes even harder if you take a look at individual the images of individual example events.

![Averages of jets](../jet_individual.png#center)

Apparently, the images are very sparse, which means only a couple of pixels are activated. In addition, considering the logarithmic scale, most pixels are activated only very weakly, meaning that the majority of the energy in the calorimeters is deposited only in a hand full of pixels, which gives small room for distinctive features. In addition, convolutional networks can struggle with very sparse structures, which can pose difficulties when building robust Autoencoders. Since the quantum circuit simulations are very demanding and my hardware access was limited, I mainly focus on a very reduces version of the dataset. I produced it by center cropping the ECAL images to about 80% and then rescaling it to 12x12. The code for this preprocessing can be found [here](https://github.com/TomMago/hep-VQAE/blob/main/notebooks/Quark-Gluon_Preprocessing.ipynb).


## Classical methods

As a very simple benchmark model I consider a convolutional autoencoder. To get a feeling for the model and training, I first tried an Autoencoder on the 40x40 ECAL dataset. I build the model similar to the one proposed in {{<cite "Finke2021">}}, which resluted in about 900k parameters. I Trained the Autoencoder on the quark jet images using [AdamW](https://www.tensorflow.org/addons/api_docs/python/tfa/optimizers/AdamW), minimizing the binary crossentropy. In general I found the training to be not so straight forward as with other image datsets. The usage of PReLU activation and AdamW seemed to help stabalize the training. As latent space size I found everything between 25-35 to be suitable.
Below are given some quark jet examples from the test set to demonstrate the reconstruction. 

![Example of autoencoder reconstruction](../4040res.png#center) 

We can now use the loss (binary crossentropy) as a discriminating variable to tag gluon jets in the test set.
Since the AE was trained on the quark jets, the average loss of a gluon jet image is expected to be higher, which labels it as anomaly.
We can therefore compute the ROC curve of the anomaly tagging. Here I obtained an AUC of about $71$\%.
Considering that {{<cite "Andrews2019">}} achived an AUC of $76$\% when training on the full 125x125 ECAL images in an supervised manner, this performance seems resonably good.
However if we train the Autoencoder on the Gluon jets instead and try to tag quark jets as anomalies, I only obtain an AUC of $29$\%.
This reflects the complexity bias mentioned before. 

![roc curves](../rocs.png#center)

To have a realistic comparasion with the quantum models, I also trained an Autoencoder on the 12x12 dataset. In this case I also only used 10k images for training and especially constrained the Autoencoder to 3k parameters. Training this Autoencoder until convergence took a couple hundret epochs and resulted in an AUC of $60$\%. In the process of optimizing the models I used the EMD from {{<cite "Komiske2019">}} to judge the reconstruction performance as it seems to capture it a little better than just the loss. Using the EMD as a loss does not seem possible as the implemention as a loss is not differentiable for TensorFlow.

# Models

I implemented different models, mainly focusing on a fully quantum model, only consisting of a single parametrized quantum circuit (PQC) and Hybrid models with several classical and quantum layers.

## Fully quantum autoencoder

My implementaion of the fully quantum autoencoder is based on </br>{{< cite "Ngairangbam2021" >}} and {{< cite "Romero2016" >}}.
The architecture consists of some data encoding and trainable PQC which acts as encoder, followed by a swap test, which fixes the non latent qubits to the value of some reference bits. This compresses the quantum state to the latent space.
A detailed description on the funcitonality of the Quantum Autoencoder can be found in my [post about it](https://www.tommago.com/posts/qae/).

![quantum autoencoder structure](../qae2.png#center)

Of course the main question is, how to upload the data and what structure of circuit to use as trainable layer. I started with a basic angle embedding, where one feature is encoded per qubit. In this encoding the $j$th feature is embedded, by rotating the $j$th qubit with $e^{-i x_j\sigma_x/2}\ket{0}$. While this ecoding is simple, it only allows a single feature per qubit, which limits the method to datasets with a small number of features, or makes it nessecary to apply other dimensionality reduction techniques first. Another alternative would be the Amplitude Encoding, however this requires very deep circuits and in prototype implementaions I found its performance in the Autoencoder to be very limited.
Therefore I drew inspiration from the data re-uploading technique proposed in </br>{{<cite "PerezSalinas2019">}}. I discuss the idea in detail in my [data re-uploading post](https://www.tommago.com/posts/drc/), however, in a nutshell arbitrary dimensional data is uploaded to a single qubit multiple times to mimic a deep neural network with a hidden layer.
In order to upload a whole image to a couple of qubits, I chose to upload the image in patches.

![upload of image in patches](../drcupload.png#center)

A patch is uploaded to a single qubit, where the pixels $x_i$ of the patch are uploaded with a parametrized unitarity $U(b_i + w_i + x_i)$ with weights and biases as introduced in the [data re-uploading](https://www.tommago.com/posts/drc/). So a 12x12 image can e.g. be devided into $3\times 3=9$ patches of the size 4x4. In this case we would upload 16 pixel values on 9 qubits respectively. In the data reuploading spirit, this circuit can be entagled and repeated multiple times to add parameters and build a deeper circuit.

I trained the model with Adam, maximizing the fidelity of the swap test. Currently the maximum AUC I achived is $56.5$\%. This result uses 5 data re-uploads which leads to 1440 parameters. I would expect this to improve with more parameters. One question I'm still investigating is the best choice of reference states. In the derivation of the Autoencoder the specific choice of reference states does not matter. However together with the data reuploading as encoder I think one needs to be careful, because if the reference states are initialized as $\ket{0}$ the fidelity is maximized if all parameters are zero, creating a pseudo solution.

## Hybrid model

The hybrid models I build are basically classical layers reducing the dimension of the data and feeding it into a PQC. The qubits get measured to obtain a classical latent space, which is reconstructed to an image by classical layers. Similar models have been proposed before, e.g. in {{<cite "9799154">}}.

![upload of image in patches](../hae2.png#center)

In order to make use of the PQC i think it makes sense to use the same encoder like for the fully quantum autoencoder. This way we can upload a larger image to the PQC and reduce the dimension down to the number of qubits. For a first implementation I e.g. reduce the dimension of the image with convolutional layers down to 9x9. I then upload the image in patches like in the fully quantum case. However this time I use a kernel size of 3 and a stride of 2 to upload the data in 16 patches. This way I can measure all 16 qubits to obtain the latent space. A smaller latent space would be too small to fully reconstruct the image.

Unfortunately training this model took very long and I was not able to train it until convergence. However I achived an AUC of $57$\% without the model converging. 

# Implementation details

When I started the project I implemented the fully quantum autoencoder in tensorflow-quantum together with cirq. However I soon moved to [pennylane](https://pennylane.ai) due to the flexibility when building quantum models and their great plugin system. When you write your code in pennylane you specify a device on which to run your quantum circuits. When simulating the circuits you can e.g. use the lightning simulator, which is a fast simulation framework.

```python
dev = qml.device('lightning.qubit', wires=TOTAL_QBITS)
``` 

The `wires` thereby specifies the number of qubits to simulate. When the code is developed in pennylane you can always switch the backend for the simulations without making any changes to the code. In principle you can also use a real quantum computer e.g. with the [pennylane qiskit](https://docs.pennylane.ai/projects/qiskit/en/latest/devices/ibmq.html) plugin. One plugin that is especially useful is the [lightning.gpu](https://docs.pennylane.ai/projects/lightning-gpu/en/latest/). It enables the simulation of the circuits on GPUs using NVIDIAs [cuQuantum framework](https://developer.nvidia.com/cuquantum-sdk), which can considerably speed up the simulation when using a larger amount of qubits.

In pennylane a circuit is build by successively applying gates to different wires, e.g. the function building the circuit for the fully quantum autoencoder:

```python
def circuit(self, params, data):

    encoder(params, data)

    qml.Hadamard(wires=total_qbits-1)
    for i in range(trash_qbits):
        qml.CSWAP(wires=[total_qbits - 1, latent_qbits + i, data_qbits + i])
    qml.Hadamard(wires=total_qbits-1)
    
    return qml.expval(qml.PauliZ(total_qbits-1))
```

This function takes parameters, which are trainable, and data, which is not trainable as arguments. In the encoder function more gates are applied to upload the data with trainable weights using e.g. Pauli X rotations `qml.RX`. Thereby `total_qubits` denotes the total number of qubits the circuit contains, `data_qubits` the number of qubits the encoder uses, `latent_qbits` the size of the latent and `trash_qubits` the number of non-latent space and there also the number of reference qubits. The circuit function can be turned into a Qnode, which is a pennylane object associated with a function that returns an expecation value. Here we measure the qubit containing the result of the swap test in the $\sigma_z$ basis as described in the [QAE post](https://www.tommago.com/posts/qae/).
When a Qnode is created you specify the differentiation method for it. 

```python
circuit_node = qml.QNode(circuit, dev, diff_method="adjoint")
```

The differentiation method describes how the gradients of the parameters are calculated. On a quantum computers the gradients of a circuit can e.g. be obtained with the [parameter shift rule](https://pennylane.ai/qml/glossary/parameter_shift.html). When simulating however I would usually use the adjoint differentiation, as it is a very fast method.

You can combine the the circuit simulation and differentiation of pennylane with your machine learning framework of choice, e.g. TensorFlow, Pytorch or Jax. When using keras e.g. pennylane already provides a class for turning a Qnode into a keras layer with
```python
weight_shapes = {"weights": (num_params,)}
qlayer = qml.qnn.KerasLayer(circuit_node, weight_shapes, output_dim=num_outputs)
```
Note that to my knowledge, this currently only works if the data passed to the qlayer is onedimensional.

# Outlook

The next step is to scale the quantum models to more parameters and loger traning on more sophisticated hardware. The developmend and first experiments were performed on retail hardware which is not suitable for larger simulations. I am curious to see if the quantum models can achive the same or even a better AUC when simulating the quantum models with the same number of parameters and training for the same amount of epochs like the classical reference model.
Furthermore one we should try to train the models on larger images as there is of course a large loss of information when scaling the images down to 12x12.

Apart from the computational bottleneck, there are still a couple of questions that are not fully answered yet. I mainly want to investigate the effects of different initializations of the reference qubits and of different latent space sizes. Furthermore one could also try other entangling schemes than the simple circular entangling topology.

Since we have tried vision transformer based architectures for supervised tasks on jet images I also implemented a quantum vision transformer. To do so i replaced the self attention layer in a simple ViT with the quantum self attention proposed in </br>{{<cite "Li2022">}}. 
By now I was not able to train the model, again, due to a lack of resources, however I hope to be able to run it in the future.

All in all it was a fun experience and I'm looking forward to see what QML will be able to achive in HEP in the future.

# References

{{< bibliography cited >}}
