# Hybrid Manager Service

This service is designed to give a rudimentary best-effort cluster management services that will transfer worker nodes between SLURM and Kubernetes based on current usage in accordance to the user configuration.

It is assumed that the DevOps engineering team has a good understanding as to what the expected use of SLURM or K8S should be within their cluster and that typically one of these servicesshould be treated with priority over the other. In addition to this it is assumed that this cluster is not constantly at a high-water mark in flux between K8S and SLURM resource needs and that the total resources minimums between SLURM and K8S can be set to be configured so that they sum up to roughly 50% of overall system capacity.

## Configuration

hms_polling: The frequency at which the service should poll.
hms_kill_jobs: Kill jobs on the non-prioritized schedulre to free up resources.
hms_service_priority: Which service to prioritize.
hms_k8s_gpu_min: The minimum number of GPUs that must remain allocated to K8S.
hms_k8s_node_min: The minimum nodes of GPUs that must remain allocated to K8S.
hms_k8s_dedicate_nodes: A list of specific nodes that will remain dedicated to K8S.
hms_slurm_gpu_min: The minimum number of GPUs that must remain allocated to SLURM.
hms_slurm_node_min: The minimum number of nodes that must remain allocated to SLURM.
hms_slurm_dedicated_nodes A list of specific nodes that will remain dedicated to SLURM.

## Workflow
### Setup

Before deploying this service the following is assumed:

1) K8S has been deployed and is managing all nodes in the cluster.
2) slurmd has been deployed and is available on all nodes in the cluster.
3) nvidia-docker has been deployed and is configured as the default runtime on all nodes in the cluster.
4) All nodes in the cluster are configured with the proper NFS mountpoints.
5) At least one login node has been configured for the SLURM cluster.

This service is also intended to be deployed as a deployment within the K8S cluster.

### Initialization

Upon initialization worker nodes will be properly allocated.

1) Verify configuration file
  a) Verify no nodes are shared between the dedicated lists
  b) Verify the sum fo both nodes minimum is equal to or less than the nodes in the cluster
  b) Verify the sum fo both gpus minimum is equal to or less than the gpus in the cluster
2) Verify slurmd is running on all nodes that K8S is aware of
3) Allocate dedicated resources
  a) De-allocate necessary K8S resources
  b) Allocate necessary slurm resources
  c) De-allocate necessary slurm resources
  d) Allocate necessary K8S resources
4) Allocate nodes to the low priority scheduler
  a) Randomly select nodes from the available pool
  b) De-allocate them from the high priority scheduler
  c) Allocate them to the low priority scheduler and repeate until node and GPU minimums are met
5) Allocate nodes to the high priority scheduler
  a) De-allocate all remaining nodes from the low priority scheduler
  b) Allocate all remaining nodes to the high priority scheduler

* Note, if the sum of the gpu_min or node_min values are to close to the available resources allocation selection may fail due to the random selection and potential for a very heterogeneous gpu environment (servers with 1, 2, 4, 8, or 16 gpus for example). * 

### Resource Detection

During runtime the service polls for available resources and for resource contraints.

In K8S this is done by polling all Pods. If a POD is detected to be in a pending state due to GPU capacity this is classified as a resource demand. Resource availability is done by polling all nodes and determining if any of them have all GPUs available.

In SLURM this is done by investigating the job queue. If jobs are pending due to GPU availability this is classified as resource demand. resource availability is done by polling all nodes and determining if any of them have all GPUs available


* Note, it is assumed that the GPU worker nodes are primarily intended to run GPU workloads. Any non-GPU workloads/jobs are ignored and assumed to be non-performance-impacting or tertiary services (monitoring, DaemonSets).

### Runtime Workflows

Because we are allocating resources by node and because there is no clean way to compare priorities between SLURM and K8S, allocation is done on a best-effort basis. It is assumed that the DevOps engineers have a decent understanding of the expected workloads and have configured the service accordingly.

#### Resources are available on the low priority scheduler

If during runtime the service determines that there are free nodes nodes within the low priority scheduler the following occurs:

1) We determine if moving any of these resources will violate the configured minimums. This is done in order starting with the available node with the most GPUs.
2) Resources are de-allocated from the low priority scheduler
3) Resources are allocated to the high priority scheduler


#### Low priority scheduler demands additional resources

1) We determine if the high priority scheduler has any free resources
2) We determine if moving any of these resources will violate the configured minimums. This is done in order starting with the available node with the most GPUs.
3) Resources are de-allocated from the high priority scheduler
4) Resources are allocated to the low priority scheduler


#### High priority scheduler demands additional resources

1) We determine if the low priority scheduler has any free resources
  a) If it does not AND the hms_kill_jobs variable is true, randomly select a node on the low priority scheduler with enough GPU resources, verify that transfering these resources will not violate any resource constraints, de-allocate that node, and kill those jobs
2) We determine if moving any of these resources will violate the configured minimums. This is done in order starting with the available node with the most GPUs.
3) Resources are de-allocated from the low priority scheduler
4) Resources are allocated to the high priority scheduler


# TODOS
1) Determine how to ensure workloads are not sparsely distributed on K8S.
2) Determine how to ensure workloads are not sparsely distributed on SLURM.
3) Determine a clean way to migrate services across nodes via K8S.
4) Determine a clean way to migrate services across nodes via SLURM.
5) Determine how to kill K8S jobs based on a priority rather than at random.
6) Determine how to kill SLURM jobs based on a priority rather than at random.
7) Determine an initialization algorithm that fairly allocates resources types while maintaining minimums, rather than being random.
8) Determine a better method to handle non-GPU Pods on K8S.
9) Determine a better method to handle non-GPU jobs on SLURM.
10) Determine a better algorithm to kill jobs.