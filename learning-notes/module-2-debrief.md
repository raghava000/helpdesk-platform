# Module 2 Debrief — Compute Layer

**Date built:** 2026-06-27
**Time invested:** ~90 minutes
**Outcome:** 3 Compute Engine VMs (jenkins-master, jenkins-slave, tools-host) provisioned inside the VPC. First SSH into a real cloud Linux machine. VMs stopped post-session for cost control.

---

## 0. The 7 conceptual questions that got asked tonight

These are the questions you SHOULD ask after every module. The answers are foundational.

### Q1. What is "ephemeral"?

Ephemeral = temporary, not permanent.

- **Ephemeral external IP** — assigned when VM starts, released when VM stops. Different IP every boot. **Free.**
- **Static external IP** — reserved permanently, yours even when VM is off. **Small monthly fee.**

Analogy: Ephemeral = hotel room number (assigned on check-in, returned on check-out). Static = your home address (yours forever).

Tonight you used ephemeral. When you stopped VMs, IPs `34.93.147.216`, `8.231.96.176`, `34.47.156.192` were released back to Google's pool.

### Q2. What is a subnet?

A subnet is a smaller slice of a VPC's IP address space.

Analogy: VPC = building. Subnet = floor of the building. Each VM = one room on one floor.

```
helpdesk-vpc (the building)
├── tools-subnet     (floor 1, IPs 10.10.0.x — 256 rooms)
└── cluster-subnet   (floor 2, IPs 10.20.0.x — 4096 rooms)
```

**Why split a VPC into multiple subnets?**
1. Organization — keep tooling separate from K8s nodes
2. Security — firewall rules can target a whole subnet
3. Routing — different subnets can have different routing
4. Scaling — small subnet for tools, big subnet for GKE pods

### Q3. "Module 2 references Module 1" — what does that mean?

Module 1 created the VPC and subnets — they live in GCP now. Module 2 created VMs, but VMs must be attached to a subnet. So Module 2 needs to *point at* Module 1's subnet.

Two ways:
1. **Hardcode the name**: `subnetwork = "tools-subnet"` — works, but if you typo you find out at apply time.
2. **Data source (what we did)**: `data "google_compute_subnetwork" "tools_subnet" { name = "tools-subnet" }` — Terraform queries GCP at plan time. If the subnet doesn't exist, plan fails fast with a clear error.

### Q4. Why again `provider.tf` in compute/?

Each folder where you run `terraform init` is its own independent Terraform working directory. Each has:
- Its own state file
- Its own provider plugin downloads
- Its own configuration

`network/` and `compute/` are unrelated to Terraform unless you connect them via modules or remote state. Both need their own `provider.tf` declaring "I use the Google provider, version ~> 5.0."

Analogy: Two apartments in the same building. Each needs its own electric meter (state) and circuit breaker (provider config). They share the building (your VPC) but the apartment setup is separate.

### Q5. Data sources concept

There are two main Terraform block types:

```hcl
# resource = "Create and manage this thing"
resource "google_compute_network" "vpc" {
  name = "helpdesk-vpc"
}

# data = "Look up this thing — don't create it, don't manage it"
data "google_compute_network" "vpc" {
  name = "helpdesk-vpc"
}
```

Same type name (`google_compute_network`), different intent.

**Use data sources when:**
1. Referencing something from another Terraform folder
2. Referencing something created manually in the console
3. Dynamically discovering values (e.g., latest Ubuntu image)
4. Looking up auto-generated values (e.g., managed service endpoint)

**Reference syntax:**
- Resources: `<TYPE>.<NAME>.<ATTR>` → `google_compute_network.vpc.id`
- Data sources: `data.<TYPE>.<NAME>.<ATTR>` → `data.google_compute_network.vpc.id`

The only difference is the `data.` prefix.

### Q6. What exactly does `main.tf` do (full walkthrough)?

See Section 4 of this debrief for the line-by-line breakdown.

### Q7. How do you decide "I need 3 VMs, ephemeral IPs, etc."?

Architecture experience, not magic. The decision chain for tonight:

| Decision | Reasoning |
|---|---|
| Need Jenkins for CI/CD | Industry standard for self-hosted pipelines |
| Master + slave split | Master = brain, slave = executor. Prevents builds from crashing the UI. Production pattern. |
| Need Nexus for artifacts | Stores built packages (NPM, JAR, Docker images) |
| Need SonarQube for code quality | Industry standard static analysis |
| Combined Nexus + SonarQube | Both need a server; combining saves cost in dev |
| **Total: 3 VMs** | jenkins-master, jenkins-slave, tools-host |
| Subnet | tools-subnet (defined in Module 1) — all tooling lives together |
| Ephemeral IPs | Need to SSH from laptop; internal IPs don't work from outside VPC |
| e2-small / e2-medium | Right-size: Jenkins is light, tools-host (Sonar) is heavier |
| Ubuntu 22.04 | Free, widely-supported, what most DevOps tooling targets |

The pattern: *what does the system need to do?* → break into roles → map each role to infra → size each piece.

You'll be making these calls yourself by Module 5. Every time I make a choice, ask "why?" — that's how architecture intuition is built.

---

## 1. The big picture

Module 1 = network (VPC + subnets + firewall). Module 2 = the boxes that live ON the network (VMs).

```
GCP Project
└── VPC: helpdesk-vpc
    └── Subnet: tools-subnet (10.10.0.0/24)
        ├── jenkins-master (10.10.0.3, ext: 34.93.147.216 — ephemeral)
        ├── jenkins-slave  (10.10.0.4, ext: 8.231.96.176)
        └── tools-host     (10.10.0.2, ext: 34.47.156.192)
```

VMs are empty Ubuntu boxes right now. Module 3 (Ansible) installs Jenkins, Nexus, SonarQube on them.

---

## 2. New tools / commands introduced

| Command | What it does |
|---|---|
| `gcloud compute ssh <vm-name> --zone=<zone>` | Auto-handles SSH keys + OS Login. Connects you to the VM. |
| `gcloud compute instances list` | Show all VMs + their status (RUNNING / TERMINATED) |
| `gcloud compute instances stop <names...> --zone=<zone>` | Power off VMs (saves ~80% of cost) |
| `gcloud compute instances start <names...> --zone=<zone>` | Power on VMs again |

**The habit:** Every session ends with `stop`. Every session starts with `start`.

---

## 3. Files you wrote

```
compute/
├── provider.tf       (16 lines — same as network/provider.tf)
├── variables.tf      (30 lines — project_id, region, zone, vpc_name, tools_subnet_name)
├── terraform.tfvars  (1 line — your project_id; gitignored)
├── main.tf           (~150 lines — the meat)
└── .terraform.lock.hcl  (auto-generated, commit it)
```

---

## 4. `main.tf` line by line

### Section 1: Data sources (look up Module 1's resources)

```hcl
data "google_compute_network" "vpc" {
  name = var.vpc_name
}

data "google_compute_subnetwork" "tools_subnet" {
  name   = var.tools_subnet_name
  region = var.region
}
```

**Plain English:**
- "GCP, find me the VPC named `helpdesk-vpc`. Stash its details so I can use its `.name` in the firewall rule below."
- "GCP, find me the subnet named `tools-subnet` in region `asia-south1`. Stash its details so I can use its `.id` when attaching VMs."

### Section 2: Locals (constants for DRY code)

```hcl
locals {
  ubuntu_image = "ubuntu-os-cloud/ubuntu-2204-lts"
  common_tags  = ["helpdesk", "devops"]
}
```

`locals` = constants scoped to this Terraform configuration. Used 3 times each below (once per VM). Change the Ubuntu version in ONE place if needed.

### Section 3: Jenkins Master VM

```hcl
resource "google_compute_instance" "jenkins_master" {
  name         = "jenkins-master"
  machine_type = "e2-small"               # 2 vCPU, 2 GB RAM
  zone         = var.zone                  # asia-south1-a
  tags         = concat(local.common_tags, ["jenkins-master"])
  #              ^ network tags for firewall targeting

  boot_disk {
    initialize_params {
      image = local.ubuntu_image           # Ubuntu 22.04
      size  = 20                           # 20 GB
      type  = "pd-balanced"                # balanced perf/cost disk
    }
  }

  network_interface {
    subnetwork = data.google_compute_subnetwork.tools_subnet.id

    access_config {
      # empty block = "give me an ephemeral external IP"
      # remove this block entirely for internal-only VM
    }
  }

  metadata = {
    enable-oslogin = "TRUE"                # SSH via Google identity
  }

  description = "Jenkins master — runs UI on :8080, controls pipelines"
}
```

**Key concepts:**
- `tags` = strings attached to the VM that firewall rules can target. `target_tags = ["jenkins-master"]` on the firewall = "only apply to VMs with this tag."
- `boot_disk` = nested block describing the OS disk. `pd-balanced` = SSD-backed, balanced price/perf.
- `network_interface { subnetwork = ... access_config {} }` = attach to subnet AND give external IP.
- `enable-oslogin = "TRUE"` = lets you SSH using your Google identity. Without it you'd manage SSH keys manually.

The other two VMs (`jenkins_slave`, `tools_host`) are nearly identical — different name, machine type, disk size.

### Section 4: Firewall rule (Jenkins UI on port 8080)

```hcl
resource "google_compute_firewall" "allow_jenkins_ui" {
  name    = "allow-jenkins-ui"
  network = data.google_compute_network.vpc.name

  allow {
    protocol = "tcp"
    ports    = ["8080"]
  }

  source_ranges = ["0.0.0.0/0"]      # from anywhere (tighten in prod)
  target_tags   = ["jenkins-master"]  # only on jenkins-master VM, not slave
}
```

**New concept: `target_tags`.** Module 1's firewall rules applied to ALL VMs in the VPC. This one applies ONLY to VMs tagged `jenkins-master`. Tag-based targeting is the production pattern — open port 8080 on Jenkins, not on every random VM.

### Section 5: Outputs

```hcl
output "jenkins_master_external_ip" {
  value = google_compute_instance.jenkins_master.network_interface[0].access_config[0].nat_ip
}
```

**Decoding the long expression:**

- `google_compute_instance.jenkins_master` — the VM resource
- `.network_interface` — its list of network interfaces
- `[0]` — the first (only) one
- `.access_config` — list of external-IP configs on that interface
- `[0]` — the first (only) one
- `.nat_ip` — the actual external IP Google assigned

Each `[0]` indexes into a list. Networks/access_configs are lists because a VM CAN have multiple interfaces or multiple external IPs in advanced setups.

---

## 5. SSH walk-through (what `gcloud compute ssh` actually does)

When you ran `gcloud compute ssh jenkins-master --zone=asia-south1-a` for the first time:

1. **Generated SSH key pair** at `~/.ssh/google_compute_engine` + `~/.ssh/google_compute_engine.pub`
2. **Uploaded public key to GCP** via OS Login service (because we set `enable-oslogin = "TRUE"`)
3. **Looked up your VM's external IP** (`34.93.147.216`)
4. **Resolved your Google identity to a Linux username** (`raghava_devops_prod_gmail_com`)
5. **Connected via SSH** to that VM with that user
6. **Created your home directory** on the VM (`/home/raghava_devops_prod_gmail_com`)
7. **Dropped you into a bash shell**

Subsequent SSH attempts skip steps 1, 2, 6 — they're cached.

When you `exit`, the SSH session closes. The VM keeps running.

---

## 6. Cost control — the most important discipline

Tonight's cost ballpark (asia-south1 pricing):

| State | Daily cost (3 VMs) | Monthly if 24/7 |
|---|---|---|
| 3 VMs running 24/7 | ~₹190/day | ~₹5,700 |
| 3 VMs running 1 hr/day, stopped rest | ~₹8/day | ~₹240 |
| 3 VMs stopped (just disks) | ~₹2/day | ~₹60 |

**The habit:** End every session with stop. Start every session with start.

```bash
# End of session
gcloud compute instances stop jenkins-master jenkins-slave tools-host --zone=asia-south1-a

# Start of next session
gcloud compute instances start jenkins-master jenkins-slave tools-host --zone=asia-south1-a
```

External IPs change after each start (ephemeral). Internal IPs (`10.10.0.x`) stay the same.

**Why VMs cost money but VPC/subnets/firewalls don't:**
- Network primitives are routing rules in software — Google's overhead is negligible
- VMs use real CPU + RAM + disk on Google's hardware — direct cost
- Disks persist when VM is stopped — small storage charge continues
- External IPs charge only when allocated to a stopped VM (or static); ephemeral on running VMs = free

---

## 7. Self-test quiz

Try before reading answers.

1. What's the difference between ephemeral and static IPs?
2. What does `enable-oslogin = "TRUE"` enable?
3. Why does `compute/` need its own `provider.tf` if `network/` already has one?
4. What's a `data` block and how does it differ from a `resource` block?
5. What do `target_tags` on a firewall rule do?
6. Why is `network_interface[0].access_config[0].nat_ip` written with two `[0]` indexes?
7. What's the difference between `gcloud compute instances stop` and `terraform destroy` for a VM?
8. When you stop a VM, what's deleted and what persists?
9. Why did we use `data` to look up the VPC instead of just hardcoding `"helpdesk-vpc"` in the firewall rule?
10. If you wanted Jenkins UI accessible from your office IP only (not the whole internet), what would you change in `main.tf`?

---

## 8. Coming next — Module 3 preview

**Configuration with Ansible.** No new infra. Just install software inside the existing 3 VMs over SSH.

You'll learn:
- Ansible inventory (telling Ansible "here are the VMs to manage")
- Playbooks (YAML files describing desired configuration)
- Modules (apt, copy, service, user, etc.)
- Idempotency (running a playbook twice = same result)
- Variables and group_vars

You'll install:
- Java + Jenkins on jenkins-master
- Java + Jenkins agent on jenkins-slave
- Docker on jenkins-slave
- Nexus + SonarQube (via Docker) on tools-host

Time estimate: ~2 hours. Cost reality: VMs running ~2 hrs = ~₹15.

Before next session:
```bash
gcloud compute instances start jenkins-master jenkins-slave tools-host --zone=asia-south1-a
gcloud compute instances list   # confirm RUNNING
```

---

## 9. Self-test answers

1. Ephemeral = assigned at start, released at stop (free). Static = reserved permanently (small monthly fee). VM gets a different ephemeral IP every time it starts.
2. `enable-oslogin = "TRUE"` lets you SSH into the VM using your Google identity (no manual SSH key management). `gcloud compute ssh` handles the keys for you automatically.
3. Each Terraform folder is an independent working directory with its own state file and provider config. `network/` and `compute/` are unrelated unless you wire them via modules or remote state. Both need their own provider declaration.
4. `resource` creates and manages a thing. `data` looks up an existing thing without managing it. Use `data` when something was created elsewhere (other Terraform, console, etc.).
5. `target_tags` make the firewall rule apply ONLY to VMs that carry the specified tag(s). Without target_tags, the rule applies to all VMs in the VPC. Tag-based targeting is how you scope rules to specific VM roles.
6. `network_interface` is a list (a VM can have multiple). `[0]` picks the first. `access_config` inside that interface is also a list (a VM can have multiple external IPs). `[0]` picks the first. Then `.nat_ip` returns the assigned IP.
7. `stop` powers off the VM but keeps it (and its disk, config, data) intact — you can restart it. `terraform destroy` deletes it entirely — gone forever. Stop = pause. Destroy = delete.
8. When stopped, the VM and its disk persist (you pay ~₹0.50/day per VM for the disk). The ephemeral external IP is released. RAM contents are lost (it's like turning off a computer). Internal IP stays the same when restarted.
9. Two reasons: (a) safety — if the VPC doesn't exist, plan fails immediately with a clear error instead of hitting "not found" at apply time; (b) flexibility — if you later rename the VPC, you only change `var.vpc_name`, not every hardcoded reference.
10. Change `source_ranges = ["0.0.0.0/0"]` to `source_ranges = ["YOUR_OFFICE_IP/32"]`. The `/32` means "exactly this one IP." For multiple IPs, list them: `["1.2.3.4/32", "5.6.7.8/32"]`.

---

## 10. Revision plan

- **Tomorrow morning, 15 min:** Take the quiz cold. Don't peek at answers.
- **Tomorrow evening, 30 min:** Re-read the 7 concept answers (Section 0). These compound.
- **Day 3, 30 min:** Try writing `main.tf` from scratch (use the rung practice from Module 1). Compare against backup.
- **Day 5, 5 min:** Explain Module 2 out loud to imaginary interviewer: "I provisioned 3 VMs in my VPC's tools subnet. Each VM is..."

By Module 5, this pattern will be muscle memory.
