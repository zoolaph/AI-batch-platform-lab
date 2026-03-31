# Converged AI Batch Platform: Slurm + Kubernetes + GPU Observability

## Master Project Plan — Source of Truth

**Last updated:** 2026-03-29
**Status:** Phase 0 — Planning Complete, Ready to Begin

---

## 1. PROJECT IDENTITY

**One-line description:**
A reproducible AI batch platform lab that compares Slurm and Kubernetes-native scheduling, adds GPU provisioning with NVIDIA GPU Operator, exposes GPU telemetry with DCGM Exporter, orchestrates distributed training, automates GPU health management, and documents the operational trade-offs and failure modes of running AI workloads as infrastructure.

**Public repo name:** `ai-batch-platform-lab`

**Article title (primary):**
*From Kubernetes Platform Engineer to AI Infrastructure: Building a Slurm + Kubernetes GPU Batch Lab*

**Article title (backup):**
*What I Learned Building an AI Batch Platform with Slurm, Kueue, GPU Operator, and Grafana*

---

## 2. WHAT THIS PROJECT PROVES

By the end, someone reading the repo should believe these seven things:

1. **You understand HPC scheduling.** You know why Slurm still dominates queued compute and how `sbatch`, `squeue`, `sacct`, partitions, QOS, and fair-share scheduling work.
2. **You understand cloud-native batch.** You know how Kubernetes handles GPUs via device plugins, how Kueue manages quotas, flavors, and preemption, and where it differs from Slurm.
3. **You can provision and observe GPUs the real way.** GPU Operator + DCGM Exporter + Prometheus + Grafana with real metrics from real hardware — not screenshots from docs.
4. **You understand distributed training operations.** You can orchestrate a multi-node PyTorch distributed job, explain NCCL, rank placement, `MASTER_ADDR`, and debug communication failures.
5. **You understand storage for AI workloads.** You know the difference between scratch, persistent, and shared storage, how checkpoints work, and what breaks when storage fills or disconnects.
6. **You think like a platform engineer.** Runbooks, failure drills, health automation, cost dashboards, and architecture decision records — not just installation steps.
7. **You can ship reproducible infrastructure.** Every component is IaC-driven, deployable from a single entry point, and documented well enough that someone else can clone and run it.

---

## 3. TECHNOLOGY STACK

### Core Infrastructure
| Component | Technology | Purpose |
|-----------|-----------|---------|
| IaC - Cloud | Terraform | Provision cloud VMs, networking, storage, GPU instances |
| IaC - Config | Ansible | Node-level provisioning, Slurm install, driver setup |
| Containerization | Docker | Container images for workloads |
| Container Registry | Cloud provider registry or self-hosted | Store training images |

### HPC Layer (Slurm)
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Job Scheduler | Slurm (slurmctld, slurmd) | Traditional HPC batch scheduling |
| Accounting | Slurm accounting (slurmdbd + MariaDB) | Job history, fair-share, resource tracking |
| GPU Support | Slurm GRES (Generic Resources) | GPU-aware scheduling in Slurm |
| Storage | NFS server | Shared $HOME, $SCRATCH, $WORK directories |

### Kubernetes Layer
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Cluster | kubeadm or k3s | Kubernetes control plane + workers |
| GPU Enablement | NVIDIA GPU Operator | Drivers, device plugin, container toolkit, DCGM |
| Batch Queueing | Kueue | Cloud-native job queueing, quotas, fair-sharing |
| Storage | NFS CSI driver + PVCs | Shared filesystem for training data and checkpoints |

### Observability Layer
| Component | Technology | Purpose |
|-----------|-----------|---------|
| GPU Telemetry | DCGM Exporter | GPU metrics: utilization, memory, temperature, ECC, XID |
| Metrics | Prometheus | Scrape and store all metrics |
| Dashboards | Grafana | Cluster ops dashboards, GPU health, cost model |
| Alerting | Prometheus Alertmanager | GPU health alerts, queue starvation alerts |

### Workload Layer
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Training Framework | PyTorch (torchrun) | Distributed training across nodes |
| Training Job | A real but small model (e.g., ResNet on CIFAR-10) | Realistic enough to stress scheduling and GPUs |
| Data | CIFAR-10 or similar small dataset | Small enough to be practical, real enough to need staging |

### Automation Layer
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Task Runner | Makefile or Taskfile | Single entry point for all operations |
| CI | GitHub Actions | Validate configs, lint IaC, run smoke tests |
| GPU Health Automation | Custom controller (bash/Python) | Watch DCGM alerts, cordon nodes, drain workloads |

---

## 4. REPO STRUCTURE

```
ai-batch-platform-lab/
├── README.md                          # Project overview, quick start, architecture diagram
├── Makefile                           # All operational targets
├── LICENSE
│
├── docs/
│   ├── ARCHITECTURE.md                # High-level architecture with diagrams
│   ├── COMPARISON.md                  # Slurm vs Kubernetes scheduling analysis
│   ├── STORAGE.md                     # Storage architecture and trade-offs
│   ├── COST_MODEL.md                  # GPU-hours, queue wait times, utilization
│   ├── FAILURE_LOG.md                 # "What I got wrong" — real debugging stories
│   ├── FUTURE.md                      # InfiniBand/RDMA, Lustre, multi-tenant, etc.
│   └── adr/                           # Architecture Decision Records
│       ├── 001-why-slurm-over-pbs.md
│       ├── 002-why-kueue-over-volcano.md
│       ├── 003-why-nfs-over-lustre.md
│       ├── 004-why-kubeadm-over-k3s.md
│       ├── 005-gpu-operator-vs-manual.md
│       └── 006-pytorch-over-other-frameworks.md
│
├── infra/
│   ├── terraform/
│   │   ├── main.tf                    # Cloud provider resources
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── gpu-nodes.tf               # GPU instance definitions
│   │   ├── networking.tf              # VPC, subnets, security groups
│   │   └── storage.tf                 # NFS server, volumes
│   │
│   └── ansible/
│       ├── inventory/
│       │   └── hosts.yml              # Dynamic or static inventory
│       ├── playbooks/
│       │   ├── common.yml             # Base OS setup, users, SSH keys
│       │   ├── nvidia-drivers.yml     # NVIDIA driver installation
│       │   ├── slurm-controller.yml   # Slurm head node setup
│       │   ├── slurm-worker.yml       # Slurm compute node setup
│       │   ├── slurm-db.yml           # Slurm accounting database
│       │   ├── nfs-server.yml         # NFS server setup
│       │   ├── nfs-client.yml         # NFS mount on workers
│       │   ├── k8s-controller.yml     # Kubernetes control plane
│       │   └── k8s-worker.yml         # Kubernetes worker nodes
│       └── roles/
│           └── ...                    # Ansible roles for each component
│
├── slurm/
│   ├── config/
│   │   ├── slurm.conf                 # Main Slurm configuration
│   │   ├── slurmdbd.conf              # Accounting daemon config
│   │   ├── gres.conf                  # GPU resource definitions
│   │   └── cgroup.conf                # Cgroup enforcement for GPUs
│   ├── jobs/
│   │   ├── single-gpu-train.sbatch    # Single GPU training job
│   │   ├── multi-gpu-train.sbatch     # Multi-GPU single-node job
│   │   ├── distributed-train.sbatch   # Multi-node distributed training
│   │   ├── gpu-burn.sbatch            # GPU stress test job
│   │   └── checkpoint-resume.sbatch   # Job with checkpoint/resume logic
│   └── scripts/
│       ├── submit-comparison-suite.sh # Submit all comparison jobs
│       └── collect-accounting.sh      # Pull sacct data for analysis
│
├── kubernetes/
│   ├── cluster/
│   │   ├── kubeadm-config.yaml        # Cluster bootstrap config
│   │   └── join-config.yaml           # Worker join template
│   ├── gpu-operator/
│   │   ├── values.yaml                # GPU Operator Helm overrides
│   │   └── gpu-test-pod.yaml          # Simple GPU validation pod
│   ├── kueue/
│   │   ├── resource-flavor.yaml       # GPU flavor definitions
│   │   ├── cluster-queue.yaml         # Cluster-wide queue config
│   │   ├── local-queue.yaml           # Namespace-level queues
│   │   └── workload-priority.yaml     # Priority classes
│   ├── jobs/
│   │   ├── single-gpu-job.yaml        # Single GPU training Job
│   │   ├── multi-gpu-job.yaml         # Multi-GPU single-node Job
│   │   ├── distributed-job.yaml       # Multi-node PyTorch distributed Job
│   │   ├── gpu-burn-job.yaml          # GPU stress test Job
│   │   └── checkpoint-resume-job.yaml # Job with checkpoint PVC
│   ├── storage/
│   │   ├── nfs-provisioner.yaml       # NFS CSI driver setup
│   │   ├── pv-training-data.yaml      # PV for dataset
│   │   ├── pv-checkpoints.yaml        # PV for model checkpoints
│   │   └── pv-scratch.yaml            # PV for scratch space
│   └── monitoring/
│       ├── prometheus-values.yaml     # Prometheus Helm overrides
│       ├── grafana-values.yaml        # Grafana Helm overrides
│       ├── dcgm-exporter-values.yaml  # DCGM Exporter config
│       ├── alerting-rules.yaml        # Prometheus alert rules
│       └── dashboards/
│           ├── gpu-cluster-overview.json
│           ├── job-queue-analytics.json
│           ├── node-health.json
│           └── cost-model.json
│
├── workloads/
│   ├── Dockerfile                     # Training container image
│   ├── train.py                       # PyTorch training script
│   ├── train_distributed.py           # PyTorch DDP training script
│   ├── data_stage.py                  # Data download and staging script
│   └── checkpoint.py                  # Checkpoint save/load utilities
│
├── automation/
│   ├── gpu-health-controller/
│   │   ├── controller.py              # Watch DCGM metrics, cordon bad nodes
│   │   ├── Dockerfile
│   │   ├── deployment.yaml
│   │   └── README.md
│   └── cost-reporter/
│       ├── reporter.py                # Calculate GPU-hours, queue wait times
│       ├── Dockerfile
│       └── cronjob.yaml
│
├── runbooks/
│   ├── gpu-node-failure.md            # GPU node goes offline
│   ├── gpu-ecc-errors.md              # ECC/XID error detected
│   ├── job-starvation.md              # Job stuck in queue
│   ├── nccl-timeout.md                # Distributed training NCCL failure
│   ├── storage-full.md                # PVC or NFS full during training
│   ├── bad-image.md                   # Container image pull failure
│   ├── metrics-gap.md                 # DCGM or Prometheus gap
│   └── checkpoint-corruption.md       # Checkpoint file corrupted or missing
│
├── chaos/
│   ├── kill-gpu-node.sh               # Simulate GPU node disappearing
│   ├── fill-storage.sh                # Fill PVC to trigger storage failure
│   ├── break-nccl.sh                  # Block NCCL port to simulate network partition
│   ├── inject-xid-error.sh            # Simulate GPU XID error via DCGM
│   ├── starve-queue.sh                # Submit enough jobs to trigger starvation
│   └── corrupt-checkpoint.sh          # Corrupt a checkpoint file mid-training
│
└── .github/
    └── workflows/
        ├── lint.yml                   # Lint Terraform, YAML, Python
        ├── validate-k8s.yml           # kubeval / kubeconform on manifests
        └── docs-check.yml            # Ensure all ADRs and runbooks exist
```

---

## 5. PHASES

---

### PHASE 0: FOUNDATION — IaC AND BASE INFRASTRUCTURE
**Goal:** Stand up the raw cloud infrastructure and prove you can create and destroy it reliably.

**What you build:**
- Terraform configs to provision: 1 control/head node (CPU), 1-2 GPU worker nodes (cheapest GPU instance available — T4 or A10G spot), 1 NFS storage node (or attached volume), networking (VPC, subnet, security group allowing SSH, inter-node traffic, NCCL ports)
- Ansible playbooks for: base OS hardening and package installation, SSH key distribution, NFS server setup and client mounts, NVIDIA driver installation on GPU nodes
- Makefile with initial targets: `make infra-up`, `make infra-down`, `make provision`, `make ssh-head`, `make ssh-worker-1`
- Directory structure for `$HOME`, `$SCRATCH`, `$WORK` on NFS

**Validation tests — Phase 0 is DONE when:**
- [ ] `make infra-up` creates all resources from zero in under 15 minutes
- [ ] `make infra-down` destroys everything cleanly with no orphaned resources
- [ ] `make provision` configures all nodes via Ansible idempotently (running it twice changes nothing)
- [ ] All nodes can SSH to each other by hostname
- [ ] NFS is mounted on all nodes at `/shared` with subdirs `home/`, `scratch/`, `work/`
- [ ] A test file written on one node is immediately visible on another via NFS
- [ ] `nvidia-smi` runs successfully on every GPU node and shows the correct GPU model
- [ ] NVIDIA driver version matches across all GPU nodes
- [ ] First ADR written: `001-why-slurm-over-pbs.md`

**Cost management note:** Use spot/preemptible instances. Tear down every night. Terraform state stays in a remote backend (S3/GCS) so you can recreate everything next session.

---

### PHASE 1: SLURM CLUSTER — TRADITIONAL HPC SCHEDULING
**Goal:** Build a working Slurm cluster with GPU-aware scheduling and job accounting.

**What you build:**
- Slurm controller (slurmctld) on the head node
- Slurm compute daemons (slurmd) on GPU worker nodes
- Slurm accounting daemon (slurmdbd) backed by MariaDB
- GRES configuration for GPU resources
- Cgroup enforcement so jobs can only see their allocated GPUs
- Partitions: `gpu` (default), `gpu-priority` (preemption enabled)
- QOS levels: `normal`, `high` (higher fair-share priority)
- Test jobs: single GPU job, multi-GPU job, job requesting specific GPU type

**Workload to run:**
- `single-gpu-train.sbatch`: trains ResNet-18 on CIFAR-10 for 5 epochs on 1 GPU
- `multi-gpu-train.sbatch`: same training on 2 GPUs using `torchrun` single-node
- `gpu-burn.sbatch`: runs `gpu-burn` stress test for 60 seconds to generate real GPU telemetry

**Validation tests — Phase 1 is DONE when:**
- [ ] `sinfo` shows all GPU nodes in state `idle` with correct GRES (e.g., `gpu:t4:1`)
- [ ] `sbatch single-gpu-train.sbatch` submits, runs, and completes successfully
- [ ] Job output confirms training ran on the correct GPU (check `CUDA_VISIBLE_DEVICES`)
- [ ] `sacct` shows the completed job with correct GPU resource usage
- [ ] Submitting a job requesting 2 GPUs when only 1 is available results in `PENDING` state with reason `Resources`
- [ ] Cgroup enforcement works: a job allocated 1 GPU cannot see other GPUs (verified with `nvidia-smi` inside the job)
- [ ] `multi-gpu-train.sbatch` runs and PyTorch correctly uses both GPUs (verify with training logs showing per-GPU throughput)
- [ ] `gpu-burn.sbatch` drives GPU utilization to 90%+ (verified with `nvidia-smi` during run)
- [ ] Fair-share works: submit 10 jobs as `user-a` and 10 as `user-b`, verify interleaved scheduling via `sacct`
- [ ] Preemption works: submit a low-priority job, then a high-priority job — low-priority gets preempted
- [ ] All Slurm configs are managed by Ansible — `make provision` can rebuild the Slurm cluster from scratch
- [ ] ADR written: `003-why-nfs-over-lustre.md`

---

### PHASE 2: KUBERNETES CLUSTER — CLOUD-NATIVE BATCH
**Goal:** Build a Kubernetes cluster with GPU Operator, Kueue, and storage, and run the same workloads.

**What you build:**
- Kubernetes cluster via kubeadm (or k3s — document the choice in an ADR)
- NVIDIA GPU Operator installed via Helm (drivers, device plugin, container toolkit, DCGM, node labeling)
- Kueue installed and configured with: ResourceFlavors for GPU types, ClusterQueue with resource quotas, LocalQueues per namespace, PriorityClasses (low, normal, high)
- NFS CSI provisioner for PersistentVolumes
- PVCs for: training data (`ReadOnlyMany`), checkpoints (`ReadWriteMany`), scratch (`ReadWriteOnce`)
- Training container image built and pushed to registry

**Workload to run:**
- `single-gpu-job.yaml`: same ResNet-18 CIFAR-10 training as Slurm, as a Kubernetes Job
- `multi-gpu-job.yaml`: same multi-GPU training, via a single Pod with 2 GPU requests
- `gpu-burn-job.yaml`: same stress test, as a Kubernetes Job

**Validation tests — Phase 2 is DONE when:**
- [ ] `kubectl get nodes` shows all nodes `Ready`
- [ ] GPU nodes have label `nvidia.com/gpu.product` with correct GPU model
- [ ] `kubectl describe node <gpu-node>` shows `nvidia.com/gpu: 1` (or correct count) in allocatable resources
- [ ] GPU Operator pods are all `Running` in `gpu-operator` namespace (driver, device-plugin, toolkit, dcgm-exporter, node-feature-discovery)
- [ ] A test pod requesting `nvidia.com/gpu: 1` can run `nvidia-smi` and see exactly 1 GPU
- [ ] `kubectl get clusterqueues` shows the queue with correct resource limits
- [ ] Submitting a Job via Kueue works — job enters `Admitted` state and runs
- [ ] Submitting more jobs than GPU quota allows — excess jobs stay in `Pending` with Kueue status `Inadmissible`
- [ ] `single-gpu-job.yaml` completes and training logs show correct GPU usage
- [ ] `multi-gpu-job.yaml` completes and training logs show both GPUs active
- [ ] Checkpoints are written to the checkpoint PVC and visible on NFS
- [ ] Training data PVC is mounted as read-only and accessible from the training container
- [ ] After job completion, `kubectl get workloads` shows correct resource consumption
- [ ] All manifests are in the repo and deployable via `make k8s-up`
- [ ] ADRs written: `002-why-kueue-over-volcano.md`, `004-why-kubeadm-over-k3s.md`, `005-gpu-operator-vs-manual.md`

---

### PHASE 3: OBSERVABILITY — METRICS, DASHBOARDS, AND ALERTING
**Goal:** Full GPU telemetry pipeline from DCGM through Prometheus to Grafana dashboards, with meaningful alerts.

**What you build:**
- DCGM Exporter already running from GPU Operator (Phase 2) — verify it exposes `/metrics`
- Prometheus installed via Helm with scrape targets for: DCGM Exporter on every GPU node, Kueue metrics, Kubernetes node metrics (node-exporter), Slurm exporter (prometheus-slurm-exporter) on the head node
- Grafana with four dashboards: **GPU Cluster Overview** (GPU utilization, memory, temperature, power draw per node), **Job Queue Analytics** (queue depth, wait times, admission rate — both Slurm and Kueue), **Node Health** (ECC errors, XID errors, GPU throttling events, NVLink status), **Cost Model** (GPU-hours consumed, per-user/per-queue breakdown, cluster utilization percentage)
- Alertmanager rules for: GPU utilization at 0% for >10 minutes (idle GPU), GPU temperature >85°C, ECC error count increasing, DCGM Exporter target down, Kueue queue depth >20 for >30 minutes, NFS mount unhealthy, PVC usage >90%

**Validation tests — Phase 3 is DONE when:**
- [ ] `curl http://<gpu-node>:9400/metrics` returns DCGM metrics including `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_GPU_TEMP`, `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL`
- [ ] Prometheus targets page shows all DCGM Exporter endpoints as `UP`
- [ ] Prometheus targets page shows Slurm exporter as `UP`
- [ ] Running `gpu-burn` causes GPU utilization on the Grafana dashboard to spike to 90%+ in real time
- [ ] GPU Cluster Overview dashboard shows per-node GPU utilization, memory, temperature — with data from real jobs
- [ ] Job Queue Analytics dashboard shows jobs submitted to both Slurm and Kueue with queue wait times
- [ ] Node Health dashboard shows zero ECC errors (baseline — will be useful in Phase 5 chaos testing)
- [ ] Cost Model dashboard shows total GPU-hours consumed over the past 24 hours
- [ ] At least one alert fires and is visible in Alertmanager (trigger by stopping DCGM Exporter on one node)
- [ ] Grafana dashboards are exported as JSON and stored in the repo under `kubernetes/monitoring/dashboards/`
- [ ] Screenshots of every dashboard with real data are saved in `docs/` for the article

---

### PHASE 4: DISTRIBUTED TRAINING — MULTI-NODE ORCHESTRATION
**Goal:** Run the same multi-node distributed training job on both Slurm and Kubernetes, and understand the operational differences.

**What you build:**
- `train_distributed.py`: PyTorch DDP training script that: uses `torchrun` for process launching, reads `MASTER_ADDR`, `MASTER_PORT`, `WORLD_SIZE`, `RANK` from environment, uses NCCL backend for GPU communication, writes checkpoints to shared storage every N steps, supports resume from checkpoint, logs per-rank throughput and loss
- Slurm version: `distributed-train.sbatch` that: uses `srun` to launch across nodes, sets NCCL environment variables, configures `torchrun` with `--nnodes`, `--nproc_per_node`, exports correct `MASTER_ADDR` from first node
- Kubernetes version: `distributed-job.yaml` that: uses a multi-worker Job (or PyTorchJob CRD from Kubeflow Training Operator if warranted — document choice), configures headless Service for rank discovery, sets GPU requests, mounts shared PVCs for data and checkpoints
- Checkpoint/resume test: start training, kill it mid-run, resume from checkpoint — on both platforms

**Validation tests — Phase 4 is DONE when:**
- [ ] Distributed training runs across 2 nodes on Slurm, each using 1 GPU, for 10 epochs — training loss decreases
- [ ] Slurm job logs show NCCL initialization: `NCCL INFO` lines showing ring/tree topology
- [ ] Same distributed training runs on Kubernetes across 2 pods on 2 nodes
- [ ] Kubernetes job logs show identical NCCL initialization and communication
- [ ] Training throughput (images/sec) is documented for both platforms — numbers are comparable
- [ ] Checkpoints are written to shared NFS on both platforms
- [ ] Kill the Slurm job mid-training → resubmit → training resumes from last checkpoint
- [ ] Kill the Kubernetes pod mid-training → Job restarts pod → training resumes from last checkpoint
- [ ] NCCL environment variables are documented: `NCCL_DEBUG`, `NCCL_SOCKET_IFNAME`, `NCCL_IB_DISABLE`
- [ ] The difference in how Slurm vs Kubernetes handles rank assignment is documented in `COMPARISON.md`
- [ ] ADR written: `006-pytorch-over-other-frameworks.md`

---

### PHASE 5: CHAOS ENGINEERING — FAILURE DRILLS AND RUNBOOKS
**Goal:** Systematically break things, document the failure modes, write incident-style runbooks, and demonstrate GPU health automation.

**What you build:**
- Chaos scripts in `chaos/` directory
- GPU health controller (`automation/gpu-health-controller/`): a Python or bash daemon that: polls DCGM Exporter metrics, detects XID errors or ECC error increase or GPU fallen off bus, cordons the Kubernetes node (`kubectl cordon`), drains running workloads, sends alert to Alertmanager, marks the Slurm node as `drain` state with reason
- Runbooks for every failure scenario

**Failure drills to execute and document:**

1. **GPU node disappears** (`chaos/kill-gpu-node.sh`)
   - Stop the GPU worker VM mid-training
   - Document: what happens to the Slurm job (killed? requeued?), what happens to the Kubernetes pod (evicted? rescheduled?), how long until the scheduler notices, what the Grafana dashboards show

2. **GPU ECC/XID error** (`chaos/inject-xid-error.sh`)
   - Simulate via DCGM injection or by manipulating exported metrics
   - Document: health controller detects and cordons node, alert fires in Alertmanager, running workloads are drained, recovery steps after GPU is healthy

3. **Job starvation** (`chaos/starve-queue.sh`)
   - Submit 20 high-priority jobs to consume all GPUs, then submit low-priority jobs
   - Document: how Slurm fair-share handles it vs how Kueue handles it, which system makes starvation more visible, how to diagnose and resolve

4. **NCCL timeout during distributed training** (`chaos/break-nccl.sh`)
   - Block NCCL port (typically 29500 or the ephemeral range) with iptables mid-training
   - Document: what the error looks like in PyTorch logs, how long before timeout, recovery steps, recommended `NCCL_TIMEOUT` settings

5. **Storage full** (`chaos/fill-storage.sh`)
   - Fill the checkpoint PVC to 100% mid-training
   - Document: what error PyTorch throws, what happens to the job, how to recover, monitoring gap if the job just hangs

6. **Bad container image** (`chaos/` — manual drill)
   - Submit a job referencing a non-existent image tag
   - Document: Kubernetes `ImagePullBackOff` behavior, Slurm equivalent if using Singularity/Enroot, time to detect, time to recover

7. **Checkpoint corruption** (`chaos/corrupt-checkpoint.sh`)
   - Write garbage to a checkpoint file between training steps
   - Document: what happens on resume, how to detect corruption, how to implement checkpoint validation

8. **Metrics gap** (`chaos/` — stop DCGM Exporter)
   - Kill DCGM Exporter pod during a training run
   - Document: Prometheus alert fires, Grafana shows gap, what the health controller does (or doesn't do), impact on operational visibility

**Validation tests — Phase 5 is DONE when:**
- [ ] Every chaos script in `chaos/` runs and produces the expected failure
- [ ] Every failure has a corresponding runbook in `runbooks/`
- [ ] Each runbook contains: symptoms, detection method, impact, resolution steps, prevention
- [ ] GPU health controller is deployed on Kubernetes and successfully: detects simulated GPU issue, cordons the affected node, drains workloads, fires an alert
- [ ] GPU health controller equivalent for Slurm: sets node to `drain` state with a descriptive reason
- [ ] The health controller has its own Dockerfile and Kubernetes deployment manifest
- [ ] `FAILURE_LOG.md` contains at least 3 real debugging stories from the project — things that genuinely broke and how you fixed them
- [ ] Grafana dashboards show visible evidence of each failure drill (screenshots saved)

---

### PHASE 6: COMPARISON ANALYSIS AND COST MODEL
**Goal:** Produce the Slurm vs Kubernetes comparison document and the cost model that makes this project stand out.

**What you build:**
- `COMPARISON.md`: structured analysis covering: job submission workflow (sbatch vs kubectl apply), scheduling model (partitions/QOS vs Kueue queues/flavors), GPU allocation mechanism (GRES vs device plugin), job visibility and debugging (squeue/sacct vs kubectl/Kueue status), preemption and fair-share, multi-node/distributed training setup complexity, storage model ($SCRATCH/$WORK vs PVCs), operational overhead (what does day-2 look like on each), failure recovery behavior (from Phase 5 drills), which is better for what (clear recommendation per use case)
- `COST_MODEL.md`: GPU-hours consumed per job type, queue wait time distribution, cluster utilization percentage over time, cost comparison: what this lab cost to run, extrapolation to 100-node cluster scale
- Cost Reporter automation (`automation/cost-reporter/`): script that queries Prometheus for GPU utilization over time, calculates GPU-hours per job from Slurm accounting + Kueue metrics, generates a simple JSON report, runs as a Kubernetes CronJob
- `STORAGE.md`: how data staging works in Slurm ($SCRATCH conventions) vs Kubernetes (PVCs), checkpoint strategy and storage requirements, what breaks when storage is slow or full (reference Phase 5), what you'd change at production scale (Lustre, GPFS, parallel filesystem)

**Validation tests — Phase 6 is DONE when:**
- [ ] `COMPARISON.md` has at least 10 dimensions of comparison with specific observations (not generic "Kubernetes is more modern")
- [ ] Every comparison point references actual behavior observed during the project, not copied from docs
- [ ] `COST_MODEL.md` contains real numbers from the lab: total cloud spend, GPU-hours consumed, cost per training run
- [ ] Cost Reporter CronJob runs daily and writes reports to shared storage
- [ ] Grafana Cost Model dashboard shows data from the cost reporter
- [ ] `STORAGE.md` documents the actual storage setup and its limitations
- [ ] All three documents explicitly state what you'd do differently at production scale

---

### PHASE 7: DOCUMENTATION, ARTICLE, AND POLISH
**Goal:** Make the repo and the public article publication-ready.

**What you build:**
- `README.md` rewrite: project purpose (2 sentences), architecture diagram (Mermaid or image), quick start (`make infra-up && make provision && make slurm-up && make k8s-up && make observe`), what's inside (table of contents), technologies used, screenshots of Grafana dashboards with real data, link to article
- Architecture diagram showing all components and data flow
- All ADRs reviewed and complete (6 ADRs minimum)
- Article draft covering: why you built this, architecture overview, the three most interesting things you learned, one thing you got wrong (from FAILURE_LOG.md), the Slurm vs Kubernetes comparison highlights, what you'd do differently at 500-node scale, links to repo
- GitHub Actions CI: lint all Terraform with `terraform validate`, lint all Kubernetes manifests with `kubeconform`, lint Python with `ruff`, check that all runbooks referenced in chaos scripts exist

**Validation tests — Phase 7 is DONE when:**
- [ ] README.md can be read in under 5 minutes and gives a clear picture of the entire project
- [ ] Architecture diagram exists and is accurate
- [ ] All 6+ ADRs are written and follow the format: context → decision → consequences
- [ ] All 8 runbooks are written and follow the format: symptoms → detection → impact → resolution → prevention
- [ ] `FAILURE_LOG.md` has at least 3 genuine debugging stories
- [ ] Grafana dashboard screenshots with real GPU data are in the repo
- [ ] GitHub Actions CI passes on main branch
- [ ] Article draft is complete and reviewed
- [ ] A fresh clone of the repo has clear instructions to reproduce the entire lab
- [ ] Someone unfamiliar with the project can read the README and understand what it proves

---

## 6. CLEAR FINISH LINE

**The project is COMPLETE when all of the following are true:**

1. The repo is public on GitHub with a clean commit history
2. All Phase 0–7 validation checkboxes are checked
3. The article is published (personal blog, Medium, Dev.to, or LinkedIn)
4. You can answer these questions cold in a 30-minute conversation:
   - "Why does Slurm still matter when Kubernetes exists?"
   - "Walk me through how a distributed training job gets scheduled on Kubernetes with Kueue"
   - "What happens when a GPU node dies mid-training in your lab?"
   - "How do you monitor GPU health at scale?"
   - "What's the difference between how Slurm and Kubernetes handle fair-share scheduling?"
   - "What would you change about this architecture at 500 nodes?"
   - "What was the hardest thing you debugged in this project?"
5. The Grafana dashboards contain real data from real GPU workloads — not synthetic or mocked data

---

## 7. MAKE TARGETS (REFERENCE)

```makefile
# Infrastructure
make infra-up          # Terraform apply — create all cloud resources
make infra-down        # Terraform destroy — tear down everything
make provision         # Ansible — configure all nodes
make ssh-head          # SSH into head/control node
make ssh-worker-1      # SSH into first GPU worker

# Slurm
make slurm-up          # Deploy Slurm cluster via Ansible
make slurm-test        # Submit test jobs and verify output
make slurm-status      # Show sinfo, squeue, sacct summary

# Kubernetes
make k8s-up            # Bootstrap Kubernetes cluster + GPU Operator + Kueue
make k8s-test          # Submit test jobs and verify output
make k8s-status        # Show nodes, GPU allocatable, Kueue queues

# Observability
make observe           # Deploy Prometheus + Grafana + DCGM Exporter + Alertmanager
make dashboards        # Import Grafana dashboards from JSON
make alerts-test       # Trigger a test alert and verify it fires

# Workloads
make train-slurm       # Submit the comparison training suite on Slurm
make train-k8s         # Submit the comparison training suite on Kubernetes
make train-distributed # Submit distributed training on both platforms

# Chaos
make chaos-gpu-fail    # Run GPU node failure drill
make chaos-nccl        # Run NCCL timeout drill
make chaos-storage     # Run storage full drill
make chaos-all         # Run all chaos drills sequentially

# Reports
make cost-report       # Generate cost/utilization report
make comparison        # Generate comparison data from both platforms

# Cleanup
make clean             # Remove all jobs, pods, temp files
make nuke              # infra-down + clean — total teardown
```

---

## 8. CLOUD COST ESTIMATE

**Per session (assuming 4-hour work blocks on spot instances):**
| Resource | Spec | Hourly Spot | Per Session |
|----------|------|-------------|-------------|
| Head node | 4 vCPU, 16GB RAM | ~$0.04 | ~$0.16 |
| GPU worker 1 | T4 instance (g4dn.xlarge or equivalent) | ~$0.16 | ~$0.64 |
| GPU worker 2 | T4 instance | ~$0.16 | ~$0.64 |
| NFS storage | 100GB SSD volume | ~$0.01 | ~$0.04 |
| **Total per session** | | | **~$1.50** |
| **Total for project (est. 40 sessions)** | | | **~$60** |

*Note: A10G instances cost ~$0.50/hr spot. Use T4 for most phases, A10G only when needed for multi-GPU or higher throughput tests.*

---

## 9. MODIFICATION LOG

| Date | Change | Reason |
|------|--------|--------|
| 2026-03-29 | Initial plan created | Project kickoff |


