# infra-case-studies

Technical narratives explaining infrastructure architecture decisions, trade-offs, and outcomes. Real-world problem-solving documented for portfolio demonstration.

## What This Demonstrates

**For Technical Leadership:**

Technical writing and decision-making skills. Ability to articulate trade-offs and justify architectural choices with data. Thoughtful infrastructure design through documented learning from production experience.

## Case Study Index

### 1. Storage Architecture Decision
**Status:** Planned (Week 2-3)  
**File:** `01-storage-architecture-decision.md`

**Problem:** Mac Mini M1 with limited internal storage needs 9TB capacity for 4-service production stack

**Solution:** Qwiizlab UH25 Pro USB enclosure + Btrfs tiered filesystem + capacity planning based on service growth projections

**Key Trade-offs:**
- USB bandwidth limitations vs capacity needs (chose capacity)
- SSD performance vs cost (hybrid tiering approach)
- Single disk simplicity vs multi-disk redundancy (single disk with Btrfs snapshots)

**Outcome:** 7.3TB usable storage with validated performance metrics, predictable I/O characteristics, cost-effective scaling path

**Target length:** 1,800-2,000 words  
**Read time:** ~10 minutes

---

### 2. Documentation as Infrastructure
**Status:** Planned (Week 2-3)  
**File:** `02-zettelkasten-documentation-pattern.md`

**Problem:** 100KB monolithic changelog file became unmaintainable, documentation drift across 50+ files, no automation possible with unstructured docs

**Solution:** Atomic unit pattern (00-01-02-03 Zettelkasten structure) with automated drift detection via cross-reference validation

**Key Trade-offs:**
- More files vs automation safety (chose scalability via atomicity)
- Monolithic docs vs atomic entries (atomic enables validation automation)
- Manual maintenance vs automated validation (automated prevents drift)

**Outcome:** Scales to 1000+ entries without performance degradation, AI-parseable structure, real-time drift detection, zero maintenance overhead

**Target length:** 1,900-2,100 words  
**Read time:** ~11 minutes

---

### 3. Disaster Recovery from First Principles
**Status:** Planned (Week 2-3)  
**File:** `03-disaster-recovery-from-first-principles.md`

**Problem:** Services need proven recovery capability, not theoretical backups that may fail when actually needed

**Solution:** Automated daily backups + tested restore scripts + documented RTO/RPO metrics + quarterly validation drills

**Key Trade-offs:**
- Backup complexity vs operational safety (safety wins, complexity managed via automation)
- Backup frequency vs storage cost (daily sufficient, RPO <24hrs acceptable for use case)
- Manual validation vs automated testing (automated quarterly validation with manual spot-checks)

**Outcome:** RTO <30min, RPO <24hrs (tested and validated against real backups), documented failure scenarios with workarounds

**Target length:** 1,700-1,900 words  
**Read time:** ~9 minutes

---

### 4. Security Hardening: Localhost-First Approach
**Status:** Planned (Week 2-3)  
**File:** `04-security-hardening-localhost-first.md`

**Problem:** Self-hosted services need security baseline without complexity of public-facing infrastructure (reverse proxies, TLS, authentication layers)

**Solution:** Localhost-only services by default, systemd sandboxing, pre-commit secret scanning, explicit reverse proxy only when external access required

**Key Trade-offs:**
- Convenience vs security (security baseline non-negotiable, convenience via SSH tunneling)
- Public vs localhost default (localhost minimizes attack surface)
- Manual security review vs automated scanning (automated scanning via detect-secrets)

**Outcome:** Zero external exposure without explicit reverse proxy configuration, systemd hardening active for all services, pre-commit validation prevents credential leaks

**Target length:** 1,300-1,500 words  
**Read time:** ~8 minutes

---

### 5. Infrastructure as Code with Ansible
**Status:** Planned (Week 2-3)  
**File:** `05-infrastructure-as-code-ansible.md`

**Problem:** Manual server setup led to inconsistent deployments, no reproducibility across environments, configuration drift between test and production

**Solution:** Ansible playbooks for complete system bootstrap (bare metal → production), role-based organization, idempotent operations, security-first defaults

**Key Trade-offs:**
- Learning curve investment vs long-term reproducibility (IaC investment pays dividends)
- Imperative scripts vs declarative playbooks (declarative scales better, easier to maintain)
- Ansible vs Terraform (Ansible for configuration management, Terraform for cloud infrastructure)

**Outcome:** 30-minute bare metal to production deployment (measured and repeatable), reproducible across environments, security-hardened by default

**Target length:** 1,400-1,600 words  
**Read time:** ~8 minutes

---

## Case Study Format

Each case study follows structured format:

### 1. Problem Statement
What challenge needed solving? Context and constraints explained. Why existing approaches insufficient.

### 2. Solution Overview
High-level approach chosen. Key technologies and patterns employed. Rationale for solution selection.

### 3. Implementation Details
Specific technical choices. Configuration examples where relevant. Integration with existing infrastructure.

### 4. Trade-offs Analysis
**Critical section:** What alternatives considered? Why rejected? What compromises made? Cost-benefit analysis.

### 5. Measured Outcomes
Quantified results where possible. Performance metrics. Operational impact. Lessons learned.

### 6. Future Considerations
Known limitations. Potential improvements. Scaling considerations. When to revisit decisions.

## Production Context

All case studies reference real infrastructure from **[fedora-asahi-srv-m1m](https://github.com/ch1ch0-FOSS/fedora-asahi-srv-m1m)** production environment:

- 4 services (Forgejo, Vaultwarden, Syncthing, Ollama)
- 9TB tiered storage (Btrfs)
- Fedora Asahi ARM64 (Mac Mini M1)
- Localhost-only security model
- Ansible-deployed infrastructure

## Target Audience

**Hiring managers and technical leadership** evaluating:
- Architectural thinking and decision-making ability
- Technical writing and communication skills
- Learning from production experience
- Ability to justify choices with data

**Not intended for:**
- Step-by-step tutorials (see repository READMEs for that)
- Generic best practices lists (focused on specific decisions)
- Tool comparisons without context

## Writing Standards

**Length targets:**
- Minimum: 1,200 words (ensures sufficient depth)
- Maximum: 2,500 words (maintains focus)
- Target: 1,500-2,000 words (optimal for portfolio)

**Structure requirements:**
- Clear problem statement (150-250 words)
- Solution overview (200-300 words)
- Trade-offs analysis (400-600 words, most critical)
- Measured outcomes (200-300 words with quantified data)

**Tone:**
- Technical but accessible
- Honest about limitations and mistakes
- Data-driven justifications
- No marketing hype or unsupported claims

## Related Infrastructure

**Production Services:** [fedora-asahi-srv-m1m](https://github.com/ch1ch0-FOSS/fedora-asahi-srv-m1m) - Infrastructure these case studies document

**IaC Implementation:** [ansible-playbooks](https://github.com/ch1ch0-FOSS/ansible-playbooks) - Referenced in Case Study #5

**DR Procedures:** [disaster-recovery](https://github.com/ch1ch0-FOSS/disaster-recovery) - Referenced in Case Study #3

## Timeline

**Week 2 (Dec 7-13):**
- Case Study #1: Storage Architecture
- Case Study #2: Documentation Pattern
- Case Study #3: Disaster Recovery

**Week 3 (Dec 14-20):**
- Case Study #4: Security Hardening
- Case Study #5: Infrastructure as Code
- Index README polish and cross-linking

**Estimated total writing:** ~8,000 words across 5 case studies

## License

MIT

---

