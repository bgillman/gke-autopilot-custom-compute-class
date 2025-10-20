
# GKE Autopilot - Custom Compute Classes
---

## Description
This repository provides a practical guide and examples for leveraging Custom Compute Classes (CCCs) in GKE Autopilot, enabling granular control over node provisioning to optimize for cost, performance, and workload-specific requirements.

--- 

### Why use GKE Autopilot's Custom Compute Classes?
In GKE Autopilot, Google automatically manages your node infrastructure, handling scaling, security, and maintenance. By default, Autopilot uses a general-purpose compute platform, which works well for many workloads. However, some workloads have specific needs that standard nodes can't meet. This is where Custom Compute Classes come in. GKE Custom Compute Classes allow you to specify custom node configurations for your workloads in GKE Autopilot. They were introduced to give you more control over the underlying compute resources while still maintaining Autopilot's hands-off management approach.

--- 

### What Are Custom Compute Classes?
Custom Compute Classes are Kubernetes custom resources you deploy to your clusters with sets of node attributes that GKE uses to configure new nodes About built-in compute classes in Autopilot clusters. They're available in GKE Autopilot and Standard modes from version 1.30.3-gke.1451000 onwards 

---

### Built-in Compute Classes
GKE provides built-in curated ComputeClasses in Autopilot clusters: Balanced (provides higher maximum CPU and memory capacity) and Scale-Out (disables simultaneous multi-threading and is optimized for scaling out). There are also two pre-installed compute classes called autopilot and autopilot-spot available on qualified clusters running GKE 1.33.1-gke.1107000 or later 

--- 

## Key Benefits
Custom Compute Classes provide hardware customization, allowing you to customize CPU architecture for your workload needs (Intel, AMD or ARM processors), configure GPUs, or address specific performance requirements like high-memory applications GKE Autopilot: [Custom Compute Classes for Different Workload Types](https://medium.com/@bgillman_83663/gke-autopilot-9aff26bbb1eb).
Additional capabilities include specific hardware configurations for workloads requiring GPUs or TPUs, default hardware configurations across deployments, optimal machine series definitions for high-performance or cost-optimized setups, and a fallback mechanism that automatically switches to preferred hardware when resources are limited

### GKE Custom Compute Classes

Custom Compute Classes provide a declarative way to influence Autopilot's node provisioning decisions, offering several key benefits:
- **Cost Optimization**: Run fault-tolerant or batch workloads on cheaper Spot VMs with a fallback to standard instances, reducing costs by up to 50%.
  
- **Performance Tuning**: Select specific, high-performance machine families like C3 for CPU-intensive tasks, ensuring your workloads run on optimal hardware.
  
- **Improved Availability**: Define fallback priorities so that if your preferred machine type or Spot VM capacity is unavailable, GKE automatically falls back to an alternative to prevent pods from being stuck in a pending state.
  
- **Operational Simplicity**: GKE automates the underlying node management. You simply define the desired compute class in your workload's manifest, eliminating the manual overhead of managing node pools and node affinities.
  
- **Hardware Flexibility**: Easily request specialized hardware, like GPUs or specific machine series, without needing to manage dedicated node pools.

### References
https://cloud.google.com/kubernetes-engine/docs/concepts/about-custom-compute-classes
https://cloud.google.com/kubernetes-engine/docs/concepts/balanced-scale-out-autopilot
https://cloud.google.com/kubernetes-engine/docs/how-to/autopilot-compute-classes

---
### Instructions

This repo has examples of various GKE CustomComputeClasses (CCC) and the Deployments that call the CCC. This is not an exhaustive list of options. Check out the reference links below for additioanl details like using CCC cluster wide or in a namespace. It assumes you have deployed your GKE Autopilot cluster and have the necessary administrative IAM roles to perform deploying artifacts to the cluster.

The GKE Autopilot Custom Compute Classes do the following:

1. ### Spot VM with nodeSelector
   * spot-vm-node-selector-class.yaml
   * spot-vm-node-selector-deployment.yaml
  
   This Deloyment & ComputeClass uses `nodeSelector` to automatically provision and managed the node pools made up of Spot instances. However, use this with caution because if GKE Autopilot cannot find or provision a Spot VM (which would carry the required label) and were to try to provision a Standard VM instead, the resulting Standard node might not get the correct label. In this scenario, your Pod would remain in a Pending state
   
2. ### Spot VM Fallback to Standard VM
   * spot-fallback-to-std-class.yaml
   * spot-fallback-to-std.deployment.yaml
  
   This Deloyment & ComputeClass is used to primarly schedule Pods to GKE Spot VM nodes, while allowing them to run on standard nodes as a fallback, this is a common-pattern for cost-sensative, fault-tolerant workloads. The annotation ` preferredDuringSchedulingIgnoredDuringExecution` in the Deployment is a means to express a preference rather than set a hard requirement. In the ComputeClass yaml, the `optimizeRulePriority` when set to true, will automatically and gradually move your workload from the lower-priority (Standard) node back to a newly available higher-priority (Spot) node to maximize cost savings.

3. ### Autopilot Spot VM
   * autopilot-spot-class.yaml
   * autopilot-spot-deployment.yaml
  
   This Deloyment & ComputeClass is most likey the best approach to take with GKE AP and using Spot VMs. Autopilot abstracts node management. To use Spot VMs, you simply apply the `nodeSelector` using the specific GKE label: `cloud.google.com/compute-class: autopilot-spot` is applied via the Deployment, Autopilot automatically provisions a Spot VM node that matches the Pod's resource requests.
   
   **Key Consideration for #3, is that this Autopilot configuration is a Spot-only requirement. If a Spot VM cannot be provisioned or is unavailable, these Pods will remain in a *Pending* state until capacity becomes available. There is no automatic fallback to a non-Spot VM**
   
4. ### High CPU Workload - Requires Spot VM
      * high-cpu-spot-class.yaml
      * high-cpu-deployment.yaml
  
   This Deloyment & ComputeClass defines a prioritized strategy for GKE Autopilot to provision nodes for workloads that require high-CPU performance at the lowest possible cost (Spot VMs), while ensuring the workload is eventually scheduled. The fallback priorities are 
      1. First choice: Provision a C3 (Compute-Optimized) Spot VM. C3 is GKE's high-performance, compute-optimized machine series, ideal for CPU-intensive tasks. Using `spot: true` makes it cost-effective.
      2. Second choice: If C3 Spot capacity is unavailable, fall back to a N2 (Standard/High-Performance) Spot VM. N2 is a strong general-purpose machine and is the next best choice for high-CPU needs.
      3. Third choice: If N2 Spot is also unavailable, fall back to an E2 (Standard/Cost-Effective) Spot VM. E2 is the default, general-purpose machine family in Autopilot.
         High Availablity is achieved if GKE cannot provision a Spot VM from the C3, N2, or E2 families, this setting instructs GKE to scale up a node anyway using any available machine type/capacity (which will likely be a standard, on-demand VM).
   
5. ###  High CPU Workload
   * high-cpu-class.yaml
   * high-cpu-deployment.yaml
   
   This Deloyment & ComputeClass provides a three-tier hardware preference designed to prioritize compute performance and then fall back for availability or cost.

   **C3 (Compute-Optimized)** - Highest Performance. GKE will first try to provision a C3 node, which is optimized for high vCPU-to-memory ratios and is ideal for CPU-intensive workloads.

   **N2 (High-Performance)** - Strong Fallback. If C3 nodes are unavailable, GKE falls back to the N2 machine family, which still offers excellent general-purpose performance.

   **E2 Spot** - Cost & Availability Fallback. As a last resort, GKE will attempt to provision a cost-effective E2 Spot VM. This balances the workload's scheduling needs with a significant cost reduction, accepting a lower performance tier and potential preemption risk.

   `whenUnsatisfiable: ScaleUpAnyway:` This ensures that even if GKE cannot provision a node that matches any of the three machine family priorities, it will still provision some available machine type (likely a standard E2) just to get the Pod scheduled. This strongly prioritizes availability and scheduling success over strict adherence to the preferred hardware.

   `autopilot: enabled: true:` Confirms that node management (including node pool creation, sizing, taints, and labels) is fully automated by GKE Autopilot.

6. ### High Memory Workload
   * high-memory-class.yaml
   * high-memory-deployment.yaml
  
   This Deloyment & ComputeClass are for a workload that is memory-intensive, requiring reliable, large amounts of RAM. Resources and Limits are defined and are the guaranteed minimum resources. The generous memory request is the primary driver for GKE Autopilot, requiring it to provision high-memory machine types to fit the Pods.

   The ComputeClass implements a clear priority order to provision nodes that are well-suited for high-memory, general-purpose workloads, while including a cost-saving fallback.

   **N2 Highmem (Standard/High-Performance)** Optimal Choice. GKE will first provision nodes from the N2 family, which are modern and powerful. Autopilot will select a high-memory configuration (Highmem) of the N2 series to satisfy the Deployment's memory request (8Gi).

   **N2D (AMD-based)** Performance Fallback. If N2 nodes are unavailable, GKE falls back to the N2D family. These are also high-performance and balanced, ensuring the workload still gets premium resources.

   **E2 Spot** Cost & Availability Last Resort. As the final technical fallback, GKE attempts to use a cost-effective E2 Spot VM. This sacrifices the high-performance machine family preference but ensures the workload gets scheduled at a potentially much lower cost, accepting the risk of preemption.

   