---
title: Infrastructure, Systems, and HPC at the Edge
---

This site is a technical log focused on **High-Performance Computing (HPC)** and **Distributed Systems**.

I explore projects that relate to the physical-digital loop: the systems required to synchronise **Digital Twins**, **Robotics**, and **Photorealistic Rendering** in real-time.

---

## 🏗️ Systems Architecture (MOCs)

### [[Distributed Systems & Infra]]
**The Backbone.** Adapting HPC reliability for edge clusters.
* **Orchestration:** Alternatives to K8s for [[Deterministic Edge Scheduling]].
* **Acceleration:** Offloading logic via [[DPUs and FPGA Co-design]].

### [[Edge AI & Robotics]]
**The Intelligence.** Shifting compute to the point of actuation.
* **Inference:** [[Zero-Copy AI Inference]] for robotic vision.
* **Swarms:** [[Distributed Robotic Control]] and collective behaviour.

### [[High-Performance Networking]]
**The Nervous System.** High-throughput, low-latency interconnects.
* **Fabric:** Adapting [[RDMA and RoCE]] for industrial 5G/6G.
* **Kernel Bypass:** Using [[eBPF and XDP]] for sub-millisecond telemetry.

### [[Rendering & Simulation]]
**The Interface.** Visualisation as a primary system input.
* **Neural Rendering:** [[Gaussian Splatting for Digital Twins]].
* **Simulation:** Building [[HPC-grade Simulation Gyms]].

---

## 🛠️ Engineering Logs (Projects)
* **[[Monistic Transport Engine]]** — WebGPU-based physics hypervisor with radiance cascades.
* **[[Project Alpha]]** — HPC-grade edge orchestrator. *(Coming Soon)*
* **[[Project Beta]]** — Low-latency vision-to-actuation pipelines. *(Coming Soon)*
* **[[Project Gamma]]** — Neural rendering for industrial twins. *(Coming Soon)*

## 📝 Recent Deep Dives
* [[The Latency Wall]] — Why cloud patterns fail the physical-digital loop. *(Coming Soon)*
* [[HPC vs Cloud]] — Scheduling and memory management analysis. *(Coming Soon)*
* [[The Zero-Copy Edge]] — Reducing overhead in sensor telemetry. *(Coming Soon)*