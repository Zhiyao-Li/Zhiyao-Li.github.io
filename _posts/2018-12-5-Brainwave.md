---
layout: article
title: MicroSoft's Brainwave
key: 20181205
lang: en
tags: [domain-specific hardware, neural network]
modify_date: 2018-12-5
---

![BrainwaveFPGA](https://blog-1256135234.cos.ap-chengdu.myqcloud.com/Brainwave/Microsoft-FPGA-21-1920x1280-960x540.jpg){:height="70%" width="70%"}

Project Brainwave, Microsoft's principal infrastructure designed for real-time AI, can ingest a request as soon as it is received over the network at high throughput and at ultra-low latency without batching. In Google's demand survey, MLP models occupy the most ratio of deployed TPUs, but as for Microsoft, text-driven scenarios are "pervasive", which means memory-intensive recurrent neural networks(LSTM and GRU) play the most part.

Project Brainwave leverages the massive high-performance FPGA infrastructure directly connected with the datacenter network as a resource pool and exposes it as hardware microsrvices. By pinning pre-trained DNN models across multiple FPGAs and pinning model parameters entirely in high-bandwidth on-chip memories, this system achieve near-peak processing efficiencies at low batch sizes.

### Catapult-enhanced servers
The Brainwave system targets Microsoft's hyperscale datacenter architecture with Catapult-enhanced servers. Each FPGA operates in-line between the server's network interface card (NIC) and the top-of-rack (TOR) switch, enabling in-situ processing of network packets and point-to-point connectivity between "hundreds of thousands" of FPGAs at low latency. Microsoft disaggregates FPGAs from CPUs to form an independent hardware microservices, which enables two critical capabilities:
  * the ability to reclaim underutilized resources for other services by rebalancing the subscription between CPUs and FPGAs
  * supporting workloads that cannot fit or run effectively on a single FPGA

![BrainwaveNetwork](https://blog-1256135234.cos.ap-chengdu.myqcloud.com/Brainwave/Brainwave_Network.PNG){:height="70%" width="70%"}

### Brainwave Architecture & Stack
* During offline compilation, the Brainwave tool flow splits DNN models into sub-graphs, each of which can fit into on-chip FPGA memory or run on the CPU.

* The heart of the Project Brainwave system is a highly efficient soft NPU hosted on each FPGA, but unlike Google's TPU, Brainwave is a system rather than a single processor unit. There is an important synergy between on-chip pinning and the integration with hardware microservices. When a single FPGA’s on-chip memory is exhausted, the system user can allocate more FPGAs for pinning the remaining parameters by scaling up a single hardware microservice and leveraging the elasticity of cloud-scale resources. This contrasts to single-chip NPUs (with no direct connectivity to other NPUs) designed to execute models stand-alone. Under these constraints, exploiting pinning would require scaling down models undesirably to fit into limited single-device on-chip memory or would require spilling of models into off-chip DRAM, increasing the single-batch serving latencies by an order of magnitude or more. 

* The Brainwave system mainly includes three layers.
  1. A tool flow and runtime for low-friction deployment of trained models, it converts pre-trained DNN models to a deployment package
  2. A distributed system architecture mapped onto CPUs and hardware microservices (network-attached FPGAs)
  3. A high-performance soft DNN processing unit synthesized onto FPGAs

  ![BrainwaveLayers](https://blog-1256135234.cos.ap-chengdu.myqcloud.com/Brainwave/Brainwave_Layer.PNG){:height="70%" width="70%"}

* A pre-trained DNN model developed in a framework is first exported into a common graph intermediate representation (IR).Tool flow optimizes the intermediate representation and partitions it into sub-graphs assigned to different CPUs and FPGAs. Device-specific backends generate device assembly and are linked together by a federated runtime that gets deployed into a live FPGA hardware microservice. The below figure shows the Brainwave tool flow and graph partitioning.

  ![BrainwaveToolFlow](https://blog-1256135234.cos.ap-chengdu.myqcloud.com/Brainwave/brainwave_toolchain.PNG){:height="70%" width="70%"}











