# Case Study #1: Storage Architecture Decision - Enhanced Version

## Problem Statement

In September 2025, the decision to build a production-capable infrastructure on a refurbished Apple Mac Mini M1 presented immediate storage constraints. The device, acquired second-hand for approximately $700 and upgraded to 16GB RAM with 512GB internal SSD storage, was initially positioned as a basic consumption-oriented machine. Community inquiries and AI-assisted research quickly revealed its limitations for anything beyond light desktop usage—particularly for workloads involving local large language models (LLMs) and quantitative trading tools, which demand significant data capacity and compute agency.

The core challenge was achieving true data ownership without vendor lock-in. Apple's ecosystem, while familiar, enforced a walled garden that prioritized SaaS subscriptions over local control. Internal storage of 512GB proved insufficient almost immediately; basic experimentation with LLMs under macOS hit roadblocks due to storage exhaustion and restrictive policies that funneled users toward cloud services like iCloud. This created a dependency on external providers, incurring ongoing costs and limiting creative freedom. The target was a self-contained system capable of handling 10TB+ of active data—driven by projected needs for model weights, datasets, and self-hosted services—while maintaining a small footprint and budget under $1,000 total.

Timeline pressure was minimal, rooted in personal motivation to escape macOS limitations rather than external deadlines. However, the rapid growth of LLM and trading workloads meant delays could hinder progress, forcing hardware upgrades or workflow compromises. Budget constraints were flexible but pragmatic: the goal was maximum agency per dollar, avoiding recurring cloud fees that could exceed hardware costs over time. Research emphasized local solutions to enable full independence, with the M1's ARM64 architecture adding compatibility challenges for Linux-based self-hosting.

## Solution Overview

The selected architecture centered on the **Qwiizlab UH25 Pro USB enclosure**, paired with dual SSDs totaling 10TB raw capacity, formatted using Btrfs on Fedora Asahi Linux. This external hub, costing around $100, houses both an 8TB primary SSD for general storage and a 2TB high-performance SSD for compute-intensive tasks, connected via USB-C to the M1 Mac Mini. The enclosure's design matches the Mac Mini's footprint, creating a seamless, stackable unit that avoids cable clutter associated with portable drives.

Transitioning from macOS to Fedora Asahi (an ARM64-optimized Linux distribution) unlocked native Btrfs support, enabling features like snapshots for backups and copy-on-write for data integrity—critical for a novice user experimenting with production services. The solution defers to industry best practices: Btrfs for its balance of features and Fedora kernel integration, SSDs for active workloads, and a single-enclosure approach for simplicity. No RAID was implemented, prioritizing capacity over redundancy in this single-user setup.

This architecture separates compute (internal 512GB SSD) from data (external 9TB usable after overhead), allowing iterative experimentation without data loss. Services like Forgejo (Git hosting) and Ollama (LLM inference) were prioritized for migration, leveraging the expanded capacity for repository growth and model storage. The total investment—$100 enclosure + ~$200 for SSDs—kept the setup under $1,000, aligning with the philosophy of owning infrastructure outright rather than renting via cloud providers.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     Mac Mini M1 (2020)                          │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  Apple M1 SoC (8-core CPU, 8-core GPU)                 │     │
│  │  16GB Unified Memory                                   │     │
│  │  512GB Internal NVMe SSD (2.5-3.5 GB/s R/W)            │     │
│  └────────────────────────────────────────────────────────┘     │
│         │                                                       │
│         │ USB-C (USB 3.1 Gen 2 - 10 Gbps / ~1.2 GB/s max)       │
│         ▼                                                       │
└─────────────────────────────────────────────────────────────────┘
         │
         │
┌────────▼──────────────────────────────────────────────────────┐
│            Qwiizlab UH25 Pro USB Enclosure                    │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  Bay 1: 8TB NVMe SSD                                 │     │
│  │  Mount: /mnt/data                                    │     │
│  │  Filesystem: Btrfs (single mode)                     │     │
│  │  Usage: General storage (services, repos, datasets)  │     │
│  │  ~850-950 MB/s sustained R/W (USB bandwidth limited) │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  Bay 2: 2TB NVMe SSD                                 │     │
│  │  Mount: /mnt/fastdata                                │     │
│  │  Filesystem: Btrfs (single mode)                     │     │
│  │  Usage: Compute-intensive (Ollama models, temp data) │     │
│  │  ~850-950 MB/s sustained R/W (USB bandwidth limited) │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                               │
│  Additional Ports: 3x USB 3.0, 1x SD card reader, 1x audio    │
└───────────────────────────────────────────────────────────────┘

Storage Hierarchy:
┌──────────────────────────────────────────────────────────┐
│ Internal SSD (512GB)                                     │
│  - Operating System (Fedora Asahi)                       │
│  - System packages and binaries                          │
│  - User home directory (configs, cache)                  │
└──────────────────────────────────────────────────────────┘
         │
         │ Data separation boundary
         ▼
┌──────────────────────────────────────────────────────────┐
│ External Storage (9TB usable)                            │
│  /mnt/data (7.3TB usable):                               │
│   - Forgejo repositories                                 │
│   - Vaultwarden database                                 │
│   - Syncthing sync folders                               │
│   - Backups and snapshots                                │
│                                                          │
│  /mnt/fastdata (1.8TB usable):                           │
│   - Ollama model storage (~70GB per model)               │
│   - Temporary compute workspaces                         │
│   - Trading tool datasets                                │
└──────────────────────────────────────────────────────────┘
```

## Performance Comparison

| Storage Type | Interface | Theoretical Max | Real-World Performance | Use Case |
|--------------|-----------|-----------------|----------------------|----------|
| **Internal M1 SSD** | NVMe PCIe 3.0 x4 | ~3,000 MB/s | 2,500-3,500 MB/s read<br>2,000-3,000 MB/s write[1][2] | Operating system, binaries, user home |
| **External USB-C (Gen 2)** | USB 3.1 Gen 2 | 10 Gbps (1,250 MB/s) | 800-950 MB/s sustained[3][4] | Primary data storage (/mnt/data, /mnt/fastdata) |
| **Cloud Storage** | Internet-dependent | 50-500 Mbps typical ISP | 6-60 MB/s (variable) | Rejected: latency, cost, dependency |

**Performance Impact Analysis:**

The USB-C connection introduces a **2-3x speed penalty** compared to internal NVMe storage. M1's internal SSD achieves 2.5-3.5 GB/s reads, while external USB 3.1 Gen 2 maxes at ~950 MB/s in real-world testing. However, this trade-off was deemed acceptable for several reasons:[3][2]

1. **Workload characteristics**: Mixed I/O patterns (Git operations, database queries, model loading) rarely saturate USB bandwidth continuously
2. **Capacity priority**: 9TB external vs 512GB internal (17x expansion) outweighs speed penalty for storage-bound workloads
3. **Cost efficiency**: $200 for 10TB external SSDs vs $500+ for internal upgrade (if even feasible on M1)
4. **Latency acceptable**: 850-950 MB/s sufficient for LLM model loading (70GB model loads in ~75-80 seconds vs ~25-30 on internal)

**Btrfs Overhead:**

Btrfs metadata consumes approximately **5-10% of total capacity** depending on configuration. With 10TB raw capacity:[5][6]
- **Metadata allocation**: ~500GB-1TB reserved for copy-on-write structures, checksums, and snapshot tracking
- **Usable capacity**: ~9TB after overhead (7.3TB on /mnt/data + 1.8TB on /mnt/fastdata)
- **Trade-off justification**: Snapshot capability and data integrity checks worth 10% capacity penalty

## Cost Breakdown

| Component | Model/Specification | Cost (USD) | Rationale |
|-----------|---------------------|-----------|-----------|
| **Mac Mini M1** | Refurbished (16GB RAM, 512GB SSD) | ~$700 | Base hardware (already owned) |
| **USB Enclosure** | Qwiizlab UH25 Pro (dual-bay) | ~$100 | Matches M1 footprint, dual SSD support, peripheral ports |
| **Primary SSD** | 8TB NVMe (brand varies) | ~$120 | General storage, Btrfs-formatted |
| **Compute SSD** | 2TB NVMe (brand varies) | ~$80 | High-speed for LLM workloads |
| **Total** | | **~$1,000** | Complete self-hosted infrastructure |

**Cost Comparison vs Alternatives:**

| Solution | Setup Cost | Recurring Cost (annual) | 5-Year TCO | Notes |
|----------|-----------|------------------------|-----------|-------|
| **Qwiizlab + SSDs (chosen)** | $1,000 | $0 | $1,000 | One-time investment, full ownership |
| **Cloud Storage (10TB)** | $0 | $600-1,200 | $3,000-6,000 | Google One ($100/mo), Dropbox Business |
| **NAS (Synology 2-bay)** | $800-1,200 | $0 | $800-1,200 | Bulkier, network-dependent, overkill for single user |
| **Internal SSD Upgrade (hypothetical)** | $1,200+ | $0 | $1,200+ | Not feasible (soldered M1 SSD), requires Apple repair |

**Budget Philosophy:**

The $1,000 total represents maximum agency per dollar: full local ownership, zero vendor lock-in, and capacity sufficient for 3-5 years of projected growth. Cloud alternatives would cost $3,000-6,000 over five years while maintaining dependency on external providers. NAS solutions add network complexity without benefit for a single-user Mac Mini setup.

## Implementation Details

Research began with broad inquiries into M1-compatible storage via Perplexity AI, web searches, and forums, focusing on options that maximized the hardware's potential without internal modifications. Cloud storage (Google Drive, Dropbox) was dismissed early due to cost dependencies and lack of ownership—estimated at $10-20/month for 10TB, creating an unsustainable cash flow burden. NAS devices like Synology or QNAP surfaced in media-focused contexts but were overlooked for this use case, as they appeared bulky and suited for static archival rather than active compute augmentation.

Portable external drives were considered but rejected for logistical hassles: managing multiple cables and devices risked disorganization, especially for a stationary Mac Mini setup. The Qwiizlab UH25 Pro emerged repeatedly in Apple hardware discussions, praised for its dual-bay design supporting up to 8TB + 2TB SSDs without bottlenecks. Comparisons against alternatives like Orico or Sabrent enclosures highlighted Qwiizlab's specialization in Apple ecosystems—better thermal management, reliable USB-C passthrough, and positive reviews for M1 compatibility. At ~$100, it undercut many competitors while offering ports for peripherals, making it the clear choice for footprint and expandability.

Filesystem selection leaned on enterprise recommendations: Btrfs was chosen over ext4 (lacking snapshots), XFS (less feature-rich for backups), ZFS (memory-intensive for the M1's 16GB RAM), or APFS (macOS-locked). Sources like Fedora documentation and Perplexity queries emphasized Btrfs's native snapshotting and copy-on-write protections, ideal for a learning environment prone to experimentation. No RAID (0, 1, 5, or 10) was pursued, as research indicated it added complexity without proportional benefits for this single-disk, non-distributed setup—risk accepted for simplicity and cost savings.

Physical setup involved formatting the SSDs via Fedora Asahi's installer, mounting them under `/mnt/data` (general) and `/mnt/fastdata` (compute), and configuring Btrfs subvolumes for services. As a Linux novice, the process included **3-5 full system reinstalls** to refine the configuration—starting from GNOME desktop, then shifting to a server-minimal base for efficiency. Each iteration validated the separation of OS from data, using commands like `mkfs.btrfs` and `mount` to establish the tiered structure. Services migrated sequentially: Forgejo first for Git repositories, followed by Ollama for LLMs, confirming accessibility via `df -h` and basic I/O tests.

## Trade-offs Analysis

The primary trade-off was **USB bandwidth limitations versus raw capacity**. Research indicated USB-C on the M1 (up to 10Gbps theoretical, ~950 MB/s real-world) introduced minor latency compared to internal NVMe speeds (2.5-3.5 GB/s), but this was deemed acceptable for mixed workloads—active compute on the 2TB SSD, bulk storage on the 8TB drive. As a novice, the penalty was acceptable because it enabled 9TB usable space immediately, avoiding the "storage cliff" that would halt LLM experimentation. Internal upgrades were infeasible: the refurbished M1's SSD was soldered, and factory repairs would exceed $500 plus downtime, far costlier than the $300 external solution.[2][3]

**Cost versus performance** balance favored SSDs exclusively over hybrid SSD/HDD setups. HDDs were relegated to archival roles (e.g., NAS-like backups), as research highlighted SSDs' superiority for frequent reads/writes in trading tools and model inference. The compromise: higher upfront cost (~$200 for 10TB SSDs versus ~$100 for equivalent HDDs) for sustained performance, justified by the goal of a responsive, agency-focused system. Btrfs's complexity over ext4's simplicity was another concession—features like snapshots enabled safe backups, but required learning curve investment. This was mitigated by deferring to Fedora defaults, reducing decision risk for a beginner.

**Reliability versus capacity** prioritized expansion over redundancy: a single enclosure accepted disk failure risk (no RAID mirroring) to maximize usable space at low cost. The accepted risk—potential total data loss—was worth it for 9TB versus 4.5TB in a RAID-1 mirror, especially with backups handled separately via the [disaster-recovery playbook](https://github.com/ch1ch0-FOSS/disaster-recovery). If budget were unlimited, the architecture would scale to higher-capacity enterprise SSDs (e.g., 16TB+ Thunderbolt enclosures at 40Gbps) without USB constraints. The most painful constraint was **knowledge gaps**: RAID and benchmarking were unfamiliar, leading to conservative choices based on "what enterprises use." This was overcome through AI-assisted research, turning inexperience into a strength by avoiding unproven paths.[4]

Budget hurt least, as the $300 solution fit comfortably; time and hardware limitations (M1's fixed internals) were more restrictive, forcing external reliance. macOS's ecosystem lock-in was the ultimate constraint, pushing the Linux migration—its conveniences sacrificed for terminal-first power, which proved transformative.

## Measured Outcomes

The final architecture delivered **9TB usable external storage** after Btrfs overhead (~10% for metadata and snapshots), configured as a 7.3TB general volume (`/mnt/data`) and 1.8TB fast volume (`/mnt/fastdata`). This exceeded the 512GB internal limit by **17x**, enabling unconstrained LLM model storage (up to 100GB+ per model) and Git repositories without pruning. Physical setup took 4-6 hours total, including multiple Fedora Asahi reinstalls—longer than expected due to novice experimentation, but each failure refined the configuration without data loss thanks to the OS-data separation.[5]

**Performance validation** occurred iteratively: no formal benchmarks (e.g., `fio` or `dd` tests) were run due to lack of familiarity, but real-world UX confirmed viability. Read/write operations for Git clones and model loading felt responsive, with no perceptible USB lag during daily use—Forgejo handled 50+ repositories smoothly, and Ollama inference on the fast SSD maintained low latency. The first service migrated was **Forgejo**, chosen for its "superpower" status in enabling local Git ownership; this validated the setup by successfully hosting repositories previously cloud-dependent, confirming mount stability via `lsblk` and service logs.

Issues encountered included initial Btrfs formatting errors during early installs (resolved by following Fedora guides) and cable management challenges with USB-C passthrough, mitigated by the enclosure's compact design. Validation relied on practical tests: filling `/mnt/data` with sample datasets to 80% capacity, monitoring via `df -hT`, and ensuring services like Syncthing synced without interruption. Total downtime across iterations was under 2 hours per reinstall, with the final server-minimal base reducing boot times to 30 seconds.

Current production workload as of November 2025:
- **4 active services**: Forgejo, Vaultwarden, Syncthing, Ollama
- **Storage utilization**: ~3.2TB of 9TB used (35% capacity)
- **Repository growth**: 50+ Git repositories totaling ~12GB
- **Model storage**: 8 Ollama models (5-70GB each) totaling ~280GB
- **Backup footprint**: ~1.1TB in Btrfs snapshots and PostgreSQL dumps

Detailed operational metrics documented in [fedora-asahi-srv-m1m production infrastructure](https://github.com/ch1ch0-FOSS/fedora-asahi-srv-m1m).

## Future Considerations

Looking back from November 2025, the novice approach—starting with a full desktop install before pivoting to server-minimal—provided valuable failure-based learning but extended setup time. With hindsight, beginning directly from Fedora Server base would accelerate the terminal-first workflow realization, skipping GNOME's conveniences that ultimately proved unnecessary. RAID remains unexplored; future iterations might add software mirroring (mdadm or Btrfs RAID-1) if data criticality increases, though current backups documented in [disaster-recovery procedures](https://github.com/ch1ch0-FOSS/disaster-recovery) suffice for single-user risk tolerance.

**Scalability** is strong: the enclosure supports drive swaps for 16TB+ SSDs as prices drop (currently ~$800-1,000 per drive), and Btrfs subvolumes allow non-disruptive expansion via `btrfs filesystem resize`. Limitations include USB's theoretical ceiling (no 40Gbps Thunderbolt without adapters) and M1's fixed internals, but these align with the low-cost, high-agency philosophy. If migrating to higher-performance infrastructure, Thunderbolt 4 enclosures (OWC Envoy Pro FX at ~2.8 GB/s) would reduce USB bandwidth penalties at 3-4x cost increase.[7]

**When to revisit this decision:**
1. Storage utilization exceeds 80% (currently at 35%)—consider 16TB SSD upgrade
2. USB bandwidth becomes bottleneck—validated via `iostat` showing consistent saturation
3. Data criticality increases—implement Btrfs RAID-1 or offsite backups
4. M1 hardware obsolescence—migrate to ARM64 server-class hardware maintaining Btrfs compatibility

The architecture's success lies in its evolution from constraint-driven necessity to a flexible foundation, demonstrating that informed deference to best practices can bridge knowledge gaps effectively. Integration with [Ansible automation](https://github.com/ch1ch0-FOSS/ansible-playbooks) ensures reproducibility if complete system rebuild required.

***

