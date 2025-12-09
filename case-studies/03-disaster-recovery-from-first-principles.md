# Case Study 3: Disaster Recovery Evolution - From Reactive to Production-Ready

## Problem Statement

In September 2025, disaster recovery for srv-m1m emerged not from planning but from necessity—the system broke three times during aggressive experimentation with minimal Fedora Asahi installations. As a Linux novice transitioning from macOS, the learning curve involved trial-and-error workflows: installing tools from GitHub repositories, then removing them via `dnf remove` commands that occasionally deleted critical system dependencies. These failures forced a fundamental architectural decision: **separate data from device**. If the operating system could be treated as disposable while preserving work-in-progress data externally, catastrophic failures became learning opportunities rather than project-ending disasters.

By late November 2025, the system had stabilized into a production stack hosting Forgejo, Vaultwarden, and Syncthing. A retrospective audit was conducted to formalize disaster recovery procedures for the professional portfolio. The expectation was to find a bloated, unoptimized mess needing a complete overhaul. Instead, the audit revealed a surprisingly robust automated system: systemd timers were executing daily backups, compression (gzip) was active, and 30-day retention policies were successfully pruning old archives. The total backup footprint was a lean 400MB—far from the "storage waste" feared.

However, the audit uncovered a **critical architectural flaw**: all backup scripts were writing to `/backup/`—a directory located on the **internal 512GB SSD**, the same drive as the operating system. This violated the core "data/device separation" principle. A drive failure or OS corruption would destroy both the production environment and the backups, creating a single point of failure that rendered the automation useless in a true disaster.

## Solution Overview

The remediation strategy focused on immediate architectural correction followed by strategic expansion. The primary action was **migrating the backup target** from the internal SSD to the `/mnt/data` external storage hub (9TB RAID-like volume). This restored the 3-2-1 backup principle's foundational layer: keeping data on different physical media than the OS.

The corrected architecture utilizes **systemd timers** to trigger three distinct backup scripts daily:
1. `config-backup.sh`: Captures `/etc` systemd units and `/usr/local/bin` scripts (~5MB)
2. `forgejo-backup.sh`: Archives Git repositories and data directories (~43MB)
3. `pg-backup.sh`: Dumps all PostgreSQL databases (Forgejo + Vaultwarden) to compressed SQL (~60KB)

**Optimization validation:**
- **Compression:** `tar.gz` and `sql.gz` formats reduce storage footprint by ~90% compared to raw snapshots
- **Retention:** Scripts enforce a strict 30-day sliding window, automatically deleting archives older than `RETENTION_DAYS=30`
- **Efficiency:** Total daily consumption is under 50MB, scaling linearly with Git repository growth, utilizing <0.01% of available external storage

To address the **offsite backup gap**, Oracle Cloud Free Tier (ARM64 VM) was selected as the strategic target. This choice aligns with the srv-m1m philosophy: maintaining ARM64 architecture consistency while leveraging enterprise-grade infrastructure at zero cost. The planned implementation involves a weekly `rsync` of the `/mnt/data/backups` directory to the Oracle VM, ensuring protection against physical hardware destruction (fire, theft) without incurring monthly cloud storage fees.

## Implementation Details

The "disaster recovery" journey evolved through three distinct phases:

**Phase 1: Reactive Survival (September 2025)**
Initial recovery relied on manual intervention. After breaking the system (e.g., deleting `systemd` by mistake), recovery meant:
1. Booting macOS recovery to repartition the drive
2. Reinstalling Fedora Asahi from USB (30-45 mins)
3. Manually mounting external drives and symlinking config files
4. Re-cloning Git repositories
*Result:* Recovery took hours; data survived only because it happened to be on external drives.

**Phase 2: Silent Automation (October-November 2025)**
Scripts were written to automate backups, but implementation details were forgotten. Systemd timers ran silently in the background.
*Audit Discovery:*
- `systemctl list-timers` showed active `pg-backup.timer` and `forgejo-backup.timer`
- `ls -lh /backup/forgejo` revealed 9 successful daily backups
- `du -sh` confirmed compression was working (43MB vs 400MB raw data)
*The Flaw:* Hardcoded `BACKUP_DIR="/backup/..."` pointed to root partition (internal SSD).

**Phase 3: Production Correction (December 1, 2025)**
The remediation process was swift and documented:
1. **Script Modification:** Updated `BACKUP_DIR` variables in all three scripts to point to `/mnt/data/backups/`
2. **Migration:** Executed `sudo mv /backup/* /mnt/data/backups/` to preserve existing recovery points
3. **Cleanup:** Removed the dangerous `/backup` directory from internal storage
4. **Validation:** Manually triggered `sudo /usr/local/bin/pg-backup.sh` to confirm permission and path correctness

This transition took less than 15 minutes but fundamentally altered the system's risk profile. It transformed a "luck-based" backup strategy (hoping the SSD doesn't fail) into a robust architectural implementation.

## Trade-offs Analysis

**Automation versus Awareness:** The decision to use systemd timers ("set it and forget it") was efficient but led to "operational blindness." Because the backups ran silently and successfully, the incorrect storage location went unnoticed for weeks. **Lesson:** Automation requires observability. Future improvements will include a dashboard or notification system (e.g., email or ntfy.sh alert) detailing backup location and size to keep the operator aware of infrastructure reality.

**Local versus Cloud-First:** Storing backups locally on `/mnt/data` prioritizes **recovery speed** (RTO) and **data sovereignty** over offsite protection. Restoring 50GB of data from a local SSD takes minutes; downloading it from cloud storage takes hours. The trade-off was accepted for the initial build, with the Oracle Cloud offsite sync planned as a secondary asynchronous layer. This "local-first" approach aligns with the project's goal of maximum agency and minimal vendor dependency.

**Compression versus Deduplication:** The current `tar.gz` approach creates full independent archives daily. While compressed, it duplicates data (10 copies of the same 40MB repo = 400MB). A deduplicating tool like `restic` or `borg` would store only changes, reducing storage by ~90% further. **Decision:** For current data volumes (<1GB total backups), the simplicity of standard `tar` archives outweighs the complexity of setting up deduplication repositories. `tar.gz` files are universally recoverable without specialized tools—a disaster recovery virtue.

**Cost versus Reliability:** The choice of **Oracle Cloud Free Tier** for offsite storage introduces platform risk (accounts can be terminated arbitrarily) in exchange for zero cost. To mitigate this, the disaster recovery strategy treats Oracle as a "bonus" layer. The primary recovery path remains the local external storage; the cloud copy protects only against total physical site destruction. If Oracle terminates the account, the system remains recoverable locally, preserving the "no vendor lock-in" philosophy.

## Measured Outcomes

**Storage Efficiency:**
- **Total Backup Size:** ~450MB (for 7 days of history across all services)
- **Compression Ratio:** ~8:1 (Raw data ~3.5GB compressed to ~450MB)
- **Impact on Capacity:** 0.005% of 9TB external storage utilized

**Recovery Objectives (Measured):**
- **RTO (Recovery Time Objective):** Proven **<45 minutes** during actual system rebuilds. Reinstalling OS (15 mins) + running Ansible bootstrap (10 mins) + restoring data from `/mnt/data` (15 mins) returns system to full production state.
- **RPO (Recovery Point Objective):** **24 hours** (Daily backups at 00:00). Acceptable for a single-user infrastructure; critical work-in-progress is pushed to GitHub more frequently, reducing effective data loss to minutes for code.

**Architecture validation:**
The `tree` command audit confirmed the "separation of concerns" hierarchy:
- `/mnt/data/backups/configs`: Infrastructure state
- `/mnt/data/backups/forgejo`: Application data
- `/mnt/data/backups/postgresql`: Database dumps
This structured approach allows granular recovery—restoring just a corrupted database without rolling back the entire file system.

## Future Considerations

**Roadmap item:** Provisioning the **Oracle Cloud ARM64 VM** is the next immediate priority. This will serve dual purposes: a `rsync` target for offsite backups and a staging environment for testing Ansible playbooks on identical architecture but different distribution (Ubuntu vs Fedora), broadening the infrastructure portfolio's technical scope.

**Testing protocol:** While restoration has been proven via forced disasters (system breaks), a **quarterly "fire drill"** is planned. This involves spinning up a virtual machine, mounting the backup drive, and executing a full restoration script to verify that `tar.gz` archives are valid and decryptable. "Schrödinger's Backup"—where the state of a backup is unknown until observed—is a risk to be eliminated.

**When to revisit:** Migration to `restic` for deduplication will be triggered when daily backup size exceeds **1GB** (currently 50MB). At that scale, full daily archives become inefficient, justifying the added complexity of incremental backup tools.
