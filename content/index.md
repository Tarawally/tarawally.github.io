---
title: Infrastructure, Systems, and HPC at the Edge
---

This site documents work on **High-Performance Computing (HPC)** and **Distributed Systems**.

The focus is on systems that bridge physical and digital environments: synchronising **Digital Twins**, **Robotics**, and **Photorealistic Rendering** in real-time.

---

## Systems Architecture

### [[Distributed Systems & Infra]]
Adapting HPC reliability patterns for edge deployments.
* **Orchestration:** Alternatives to K8s for [[Deterministic Edge Scheduling]]
* **Acceleration:** Offloading logic via [[DPUs and FPGA Co-design]]

### [[Edge AI & Robotics]]
Shifting compute closer to actuation points.
* **Inference:** [[Zero-Copy AI Inference]] for robotic vision
* **Swarms:** [[Distributed Robotic Control]] and collective behaviour

### [[High-Performance Networking]]
High-throughput, low-latency interconnects for edge systems.
* **Fabric:** Adapting [[RDMA and RoCE]] for industrial 5G/6G
* **Kernel Bypass:** Using [[eBPF and XDP]] for sub-millisecond telemetry

### [[Rendering & Simulation]]
Visualisation and simulation infrastructure.
* **Neural Rendering:** [[Gaussian Splatting for Digital Twins]]
* **Simulation:** Building [[HPC-grade Simulation Gyms]]

---

## Engineering Logs
* **[[Monistic Transport Engine]]** — WebGPU-based physics hypervisor with radiance cascades

## Recent Notes
