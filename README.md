# KSM Optimization for NUMA-Aware OpenStack Compute Hosts

## Overview
This project documents the tuning and automation of Kernel Same-page Merging (KSM) for OpenStack compute nodes running on NUMA-aware architectures. By enabling and configuring KSM properly, we achieved significant RAM savings across multiple hosts running cloned virtual machines, while preserving NUMA locality for optimal performance.

---

## Goal
- Reduce memory usage across compute nodes.
- Increase VM density without sacrificing performance.
- Ensure NUMA-safe operation using `merge_across_nodes = 0`.

---

## What is KSM?
KSM is a Linux kernel feature that scans memory for identical pages and merges them using copy-on-write (COW). This is especially effective when running multiple similar or cloned VMs (e.g., training environments).

Key Benefits:
- Transparent to guests
- Saves significant memory in idle/similar VM workloads

---

## Why KSM Matters Here
In our OpenStack environment:
- Each compute node hosts 100–150 VMs
- Each node has 500 GB of RAM
- Many VMs are clones of the same image (classroom or exam template)
- Idle memory pages are often identical

We have encountered memory exhaustion and swap usage spikes during large-scale deployments. Some training sessions deploy up to 14 VMs per project, with 12 or more projects running concurrently, totaling 168 VMs or more per training wave. OpenStack starts all VMs simultaneously, leading to near memory exhaustion.

To mitigate this:
- The OpenStack RAM overcommit ratio was reduced from 1.0 to 0.85
- A custom script was developed to stop VMs one hour after deployment to reclaim memory in underused environments

This environment is a perfect scenario for KSM, which has delivered up to 30–36 GB of RAM savings per node.

---

## KSM Tuning Goals
| Setting | Purpose |
|--------|---------|
| `merge_across_nodes = 0` | Preserve NUMA-locality to prevent cross-node memory access |
| `sleep_millisecs = 100` | Reduce CPU load while scanning |
| `run = 1` | Enable the KSM engine |

---

## Installation Steps (One-Time Setup per Host)

### 1. Create the KSM tuning service:
```bash
sudo nano /etc/systemd/system/ksm-tuned.service
```
Paste the following:
```ini
[Unit]
Description=Enable and Tune Kernel Same-page Merging (KSM) with NUMA-safe settings
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c '
  echo 0 > /sys/kernel/mm/ksm/run && \
  echo 0 > /sys/kernel/mm/ksm/merge_across_nodes && \
  echo 100 > /sys/kernel/mm/ksm/sleep_millisecs && \
  echo 1 > /sys/kernel/mm/ksm/run'

[Install]
WantedBy=multi-user.target
```

### 2. Enable and start the service:
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable ksm-tuned
sudo systemctl start ksm-tuned
```

---

## Verification
Run the following to check KSM state:
```bash
cat /sys/kernel/mm/ksm/run
# Expected: 1

cat /sys/kernel/mm/ksm/merge_across_nodes
# Expected: 0

cat /sys/kernel/mm/ksm/sleep_millisecs
# Expected: 100

watch -n 10 cat /sys/kernel/mm/ksm/pages_sharing
# Monitor merged page count
```

Multiply merged pages by 4KB to estimate memory savings.
Example:
```
7,000,000 pages × 4KB = ~26.7 GB saved
```

---

## NUMA Awareness
By disabling `merge_across_nodes`, we:
- Prevent cross-node merging
- Maintain memory locality for pinned CPUs/VMs
- Avoid unpredictable performance degradation

---

## Results (Production Metrics)
| Host         | Pages Sharing   | Estimated RAM Saved |
|--------------|------------------|----------------------|
| os-compute1  | ~7 million       | ~26.7 GB             |
| os-compute2  | ~8.2 million     | ~31.3 GB             |
| os-compute3  | ~9 million       | ~34.4 GB             |

Total savings: ~92 GB across 3 compute nodes

---

## Notes
- This solution is non-intrusive and does not require guest agent or changes.
- CPU overhead is minimal when tuned with `sleep_millisecs = 100`.
- Transparent HugePages (THP) may reduce KSM effectiveness. Optional optimization could be to disable THP in the future.


---

## Conclusion
This KSM tuning guide provides an easy, production-safe way to optimize memory usage in NUMA-aware compute hosts. With cloned VMs and static workloads, KSM can significantly improve VM density with no performance tradeoff.

Use KSM + NUMA tuning to scale efficiently and save resources — especially in training and virtual lab environments.

