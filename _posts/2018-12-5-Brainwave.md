---
layout: article
title: MicroSoft Brainwave Project
key: 20181205
lang: en
tags: [domain-specific hardware, neural network]
modify_date: 2018-12-5
---

![BrainwaveFPGA](https://blog-1256135234.cos.ap-chengdu.myqcloud.com/Brainwave/Microsoft-FPGA-21-1920x1280-960x540.jpg){:height="70%" width="70%"}

[Project Brainwave](https://www.microsoft.com/en-us/research/uploads/prod/2018/03/mi0218_Chung-2018Mar25.pdf), Microsoft's principal infrastructure designed for real-time AI, can ingest a request as soon as it is received over the network at high throughput and at ultra-low latency without batching. In Google's demand survey, MLP models occupy the most ratio of deployed TPUs, but as for Microsoft, text-driven scenarios are "pervasive", which means memory-intensive recurrent neural networks(LSTM and GRU) play the most part.

Project Brainwave leverages the massive high-performance FPGA infrastructure directly connected with the datacenter network as a resource pool and exposes it as hardware microsrvices. By pinning pre-trained DNN models across multiple FPGAs and pinning model parameters entirely in high-bandwidth on-chip memories, this system achieve near-peak processing efficiencies at low batch sizes.

<!--more-->

### Catapult-enhanced servers
The Brainwave system targets Microsoft's hyperscale datacenter architecture with Catapult-enhanced servers. Each FPGA operates in-line between the server's network interface card (NIC) and the top-of-rack (TOR) switch, enabling in-situ processing of network packets and point-to-point connectivity between "hundreds of thousands" of FPGAs at low latency. Microsoft disaggregates FPGAs from CPUs to form an independent hardware microservices, which enables two critical capabilities:
  * the ability to reclaim underutilized resources for other services by rebalancing the subscription between CPUs and FPGAs
  * supporting workloads that cannot fit or run effectively on a single FPGA

![BrainwaveNetwork](https://blog-1256135234.cos.ap-chengdu.myqcloud.com/Brainwave/Brainwave_Network.PNG){:height="80%" width="80%"}

### Brainwave Architecture & Stack
* During offline compilation, the Brainwave tool flow splits DNN models into sub-graphs, each of which can fit into on-chip FPGA memory or run on the CPU.

* The heart of the Project Brainwave system is a highly efficient soft NPU hosted on each FPGA, but unlike Google's TPU, Brainwave is a system rather than a single processor unit. There is an important synergy between on-chip pinning and the integration with hardware microservices. When a single FPGA’s on-chip memory is exhausted, the system user can allocate more FPGAs for pinning the remaining parameters by scaling up a single hardware microservice and leveraging the elasticity of cloud-scale resources. This contrasts to single-chip NPUs (with no direct connectivity to other NPUs) designed to execute models stand-alone. Under these constraints, exploiting pinning would require scaling down models undesirably to fit into limited single-device on-chip memory or would require spilling of models into off-chip DRAM, increasing the single-batch serving latencies by an order of magnitude or more. 

* The Brainwave system mainly includes three layers.
  1. A tool flow and runtime for low-friction deployment of trained models, it converts pre-trained DNN models to a deployment package
  2. A distributed system architecture mapped onto CPUs and hardware microservices (network-attached FPGAs)
  3. A high-performance soft DNN processing unit synthesized onto FPGAs

  ![BrainwaveLayers](https://blog-1256135234.cos.ap-chengdu.myqcloud.com/Brainwave/Brainwave_Layer.PNG){:height="80%" width="80%"}

* A pre-trained DNN model developed in a framework is first exported into a common graph intermediate representation (IR).Tool flow optimizes the intermediate representation and partitions it into sub-graphs assigned to different CPUs and FPGAs. Partitioned sub-graphs are then passed down to device-specific backend tools. Device-specific backends generate device assembly and are linked together by a federated runtime that gets deployed into a live FPGA hardware microservice. The below figure shows the Brainwave tool flow and graph partitioning.

  ![BrainwaveToolFlow](https://blog-1256135234.cos.ap-chengdu.myqcloud.com/Brainwave/brainwave_toolchain.PNG){:height="80%" width="80%"}

  The partition approaches are employed depending on layer type and degree of optimization. **CNNs**, due to their high computational intensity, are typically mapped to single FPGAs, while matrix weights of bandwidth-limited **RNNs** are pinned in a greedy fashion across multiple FPGAs during depth-first traversal of the DNN graph. **Large matrices** that cannot fit in a single device are sub-divided into smaller matrices.

* Operators that are unsupported or not profitable for offload are grouped into sub-graphs assigned to CPUs. This heterogeneous approach preserves operator coverage by **leveraging CPUs as a catch-all** for currently unsupported or rarely exercised operators.

* The Brainwave system targets **a soft ISA**. Instead of synthesizing soft logic from a neural network description directly, the FPGA backend tool targets a custom-built processor exposing a specialized instruction set for DNNs.

* A federated **runtime** and scheduler consumes the device-specific executables for deployment. The federated runtime is responsible for integrating different runtime DLLs into a single executable, and executing sub-graphs scheduled to run on a specific runtime/device combination. The runtime also performs the marshalling of data between CPU/FPGA devices.

### Brainwave NPU
Brainwave NPU is the heart of the entire Brainwave Project, which was introduced in [A Configurable Cloud-Scale DNN Processor for Real-Time AI](https://www.microsoft.com/en-us/research/uploads/prod/2018/06/ISCA18-Brainwave-CameraReady.pdf) presented on 18 ISCA.

The Brainwave NPU's salient features:
* The use of compile-time narrow precision data types to extract higher performance than waht is possible using conventional float and integer types without losses in accuracy;
* A simple, single-threaded programming model with an extensible ISA that can adapt to fast-changing DNN algorithms;
* A scalable microarchitecture that maximizes hardwae efficiency at low batch sizes.

Systems optimized for batch throughput, like GPGPU, typically can apply only a fraction of their resurces to a single request. But in an online inference setting, **requests often arrive one at a time**, so a throughput architecture must either process these requests individually, leading to reduced throughput while still sustaining batch-equivalent latency, or incur increased latency by waiting for multiple request arrivals to form a batch.

Rather than driving up throughput at the expense of latency by exploiting inter-request parallelism, Brainwave NPU reduces latency by extracting as much parallelism as possible from individual requests.

#### Architecture
The BW NPU implements a **single-threaded SIMD ISA** comprised of matrix-vector and vector-vector operations. While using a single-threaded model for a massively parallel accelerator may seem counterintuitive, the BW NPU is able to extract sufficient SIMD and pipeline parallelism to provide high utilization from individual requests.

#### Memory system
To minimize latency, the BW system typically pins DNN model weights in **distributed on-chip SRAM memories**, which provide terabytes per second of bandwidth at low power. But Mircosoft's FPGAs also have **local DRAM** for more computationally intensive models such as CNNs.

#### Microarchitecture
The BW NPU microarchitecture uses three techniques to extract parallelism:
  * The first is **hierarchical decode and dispatch** (HDD), where compound SIMD operations are successively expanded into larger numbers of primitive operations and fanned out to the functional units.
  * The BW NPU employs **vector-level parallelism** (VLP), where the compound operations are broken into operations with a fixed native vector size and then these vector operations are fanned out to the compute units.
  * The BW toolchain maps these parallelized vector operations to a **flat, one-dimensional network of vector functional units** and connects them in a dataflow manner so vectors can flow directly from one functional unit to another to minimize pipeline bubbles.

TBC.












