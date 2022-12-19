---
title: "NERSC Open Hackathon 2022 | Multi-GPU quantum circuit simulation in Pennylane"
date: 2022-12-18
tags: ["QML"]
author: "Tom Magorsch"
ShowToc: true
bibFile: content/posts/bib_lit.json
math: true
---

The past month I have been participating in the [NERSC Open Hackathon](https://www.openhackathons.org/s/siteevent/a0C5e000005UNW4EAO/se000137) hosted together with NVIDIA.
Throughout the event we had access to the [Perlmutter compute system](https://perlmutter.carrd.co) and worked together with mentors on scaling our scientific software projects on GPUs.
During the event I worked on scaling the training of VQCs in [Pennylane](https://pennylane.ai) to multiple GPUs.
A word of thanks goes to the organizers and all the mentors who helped us throghout the event.

# Scaling quantum simulations

During my [GSoC project](https://www.tommago.com/posts/gsoc/) I worked on training QML models for HEP analysis tasks.
In the process, a major bottleneck in validating current QML models were the long training times and the inability of simulating larger qbit systems. My goal for the hackathon was therefore to
1. Enable scaling to larger qbit systems
2. Enable faster training for medium sized qbit systems

The simulation of quantum circuits can be performed on GPUs, with a promising implementation beeing the [NVIDIA cuQuantum SDK](https://developer.nvidia.com/cuquantum-sdk).
In previous benchmarks I found cuQuantum outperforming the simulation on comparable CPUs for qbit numbers of approximately $N\geq 20$. 
An especially interesting feature is provided by the [cuQuantum Appliance](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/cuquantum-appliance), as it is able to handle the splitting of the state vector across gpus. 
Since the state vector of an $N$ qubit system scales with $2^N$, it quickly supasses the memory of current GPUs when moving beyond $N=30$.
Therefore splitting it across multiple gpus becomes mandatory to simulate larger systems. 

# Setup: Pennylane + Cirq + cuQuantum

Up until now I have been using the [PennyLane-Lightning-GPU Plugin](https://docs.pennylane.ai/projects/lightning-gpu/en/latest/index.html). This plugin is build on cuQuantum and conveniently enables to execute the circuits on the GPU.
However, `lightning.gpu` has a limited multi-GPU support, as it only enables the execution of different measurements in a circuit on multiple GPUs. Especially it does not support the splitting of the state vector. Manual methods like [circuit cutting](https://pennylane.ai/qml/demos/tutorial_quantum_circuit_cutting.html) can circumvent this, however I was looking for a more generic technical solution.

The solution we came up with during the hackathon was to use the cuQuantum Appliance, which supports the scaling to multiple GPUs with cirq, together with the [PennyLane-Cirq Plugin](https://docs.pennylane.ai/projects/cirq/en/latest/) to integrate it with pennylane.
To set up your code this way you have to:
1. Run the [cuQuantum Appliance](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/cuquantum-appliance)
2. Add pennylane and pennylane-cirq with `pip install pennylane pennylane-cirq`
3. Modify your code with
```python
import qsimcirq
import pennylane as qml

qs_opts = qsimcirq.QSimOptions(cpu_threads=64, verbosity=0, gpu_mode=4)
qs = qsimcirq.QSimSimulator(qs_opts)

dev1 = qml.device('cirq.simulator', wires=NUMBER_QBITS, simulator=qs)
```
4. Done! You can run your job. However during our work we encountered a bug which resulted in the consoled beeing flooded, so you might consider redirecting the error output with `python yourscript.py 2>/dev/null`. Of course you will loose all real errors like that. 

In this example `gpu_mode` describes the number of gpus to use. In Perlmutter there are four GPUs in a node. Unfortunately to this date, the current release of the cuQuantum Appliance only supports a single node, however a multi-node version already exists as shown in [NVIDIAs Blog](https://developer.nvidia.com/blog/accelerating-quantum-circuit-simulation-with-nvidia-custatevec/) and is said to release to the public soon.

# Benchmark

To validate this is working I used a simple sample script similar to </br>[pennylane's gpu benchmark](https://pennylane.ai/blog/2022/07/lightning-fast-simulations-with-pennylane-and-the-nvidia-cuquantum-sdk/)
```python
import qsimcirq
import pennylane as qml
from timeit import default_timer as timer

wires = 28
layers = 1
num_runs = 50
GPUs = 4

qs_opts = qsimcirq.QSimOptions(cpu_threads=64, verbosity=0, gpu_mode=GPUs)
qs = qsimcirq.QSimSimulator(qs_opts)

dev = qml.device('cirq.simulator', wires=wires, simulator=qs)

@qml.qnode(dev)
def circuit(parameters):
    qml.StronglyEntanglingLayers(weights=parameters, wires=range(wires))
    return [qml.expval(qml.PauliZ(i)) for i in range(wires)]

shape = qml.StronglyEntanglingLayers.shape(n_layers=layers, n_wires=wires)
weights = qml.numpy.random.random(size=shape)

timing = []
for t in range(num_runs):
    start = timer()
    val = circuit(weights)
    end = timer()
    timing.append(end - start)
    print('run: ', t, end="\r")

print(qml.numpy.mean(timing))
```

As you can see I only benchmark the _execution of circuits_.

Below I show the results of the benchmark code for different numbers of GPUs.

![Plot of benchmark](../nersc_bench.png#center)

As we can see, this setup has only has a speedup for system above approximately $28$ qubits.
Below that, the overhead from splitting the circuit and distributing it between GPUs is too large.
I only show systems up to $32$, since larger systems don't fit into a single GPU anymore anyways.

When training VQCs we don't just want to execute circuits, but rather have to differentiate them with respect to the trainable parameters.
The by far fastest method is the _adjoint differentiation_ introduced in {{<cite "Jones2020">}}.
This method is only applicable to simulation, where it takes advantage of the fact, that we can access the state vector at any point in the circuit. When calculating the derivative with respect to many different parameters we then don't have to simulate the whole circuit every time, but can 'undo' a small number of gates by applying the adjoint and in this way get the derivative to different parameters with a very small effort.

Unfortunately this method of differentiation is not implemented in cirq, but only [here](https://github.com/PennyLaneAI/pennylane-lightning-gpu/blob/main/pennylane_lightning_gpu/src/algorithms/AdjointDiffGPU.hpp) in Pennylane-Lightning-GPU.

Since the adjoint differentiation is magnitudes faster for larger amounts of trainable parameters, currently a single GPU with adjoint differentiation method is way faster than multi-GPU with e.g. [parameter-shift](https://pennylane.ai/qml/glossary/parameter_shift.html).

# Outlook

With regards to our goals, we were able to 
1. Train larger qbits systems by splitting the state vector across gpus
However,
2. The training of medium sized qbits system is currently faster on a single due to the lack of _adjoint differentiation_ in pennylane-cirq. 

Therefore _the implementation of adjoint differentiation for cirq would be key to enable the scaling of VQC training in pennylane_.

In addition, I am curious how the multi-node Appliance will perfom. Since the dirstribution across GPUs already has a sizable overhead, I expect the splitting across nodes to be even more expensive. Nevertheless, NVIDIA [already demonstrated](https://developer.nvidia.com/blog/accelerating-quantum-circuit-simulation-with-nvidia-custatevec/) that the multi-node scaling still results in a worthwile speedup.
 
# References

{{< bibliography cited >}}
