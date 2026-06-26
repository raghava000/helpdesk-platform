# Module 1 Debrief — Helpdesk Platform DevOps Project

**Date built:** 2026-06-27 (early hours)
**Time invested:** ~3 hours
**Outcome:** Real cloud infrastructure provisioned on GCP via Terraform — VPC, 2 subnets, 2 firewall rules.

---

## 0. Why this document exists

Tonight you ran ~30 commands and wrote ~80 lines of Terraform code. You honestly admitted you copy-pasted without understanding. This document fixes that — it walks through every command, every file, every line, in plain English. Read it tomorrow when you're fresh. It is your textbook for tonight's work.

---

## 1. The big picture (what we built and why)

You're building an end-to-end DevOps project called **Helpdesk Platform** — a microservices SaaS that demonstrates a complete DevOps pipeline. The infrastructure has to exist BEFORE we can install Jenkins, deploy Kubernetes, or do anything else.

Module 1 is the **foundation layer** — the network on which everything will run.

Think of it like building a house:
- Module 1 (tonight) = laying the foundation, pouring the slab
- Module 2 (next) = framing the walls (VMs for Jenkins + tools)
- Module 3 = wiring + plumbing (Ansible installs software inside the VMs)
- Module 4 = adding rooms (Kubernetes cluster with all your services)
- Module 5 = security system + lights (observability, monitoring)

You can't skip the foundation.

---

## 2. The exact tools you installed

| Tool | What it does | Verification command |
|---|---|---|
| **Homebrew** | Mac package manager — installs software for you | `brew --version` |
| **Terraform 1.9.8** | Provisions cloud infra from code | `terraform -version` |
| **gcloud CLI 530.0.0** | Talks to Google Cloud from your terminal | `gcloud --version` |

You hit one problem during install — `brew install terraform` failed because HashiCorp moved Terraform off Homebrew (license change, 2023). The fix was downloading the binary directly from `releases.hashicorp.com`. The lesson: when brew fails, the official binary download always works.

---

## 3. Every `gcloud` command, in order, explained

Think of `gcloud` as the terminal version of clicking around the GCP web console.

### 3.1 Authentication

```bash
gcloud auth login
```
**What it does:** Opens a browser. You log in with `raghava.devops.prod@gmail.com`. The terminal remembers you. From now on every `gcloud` command runs as your Google identity.

```bash
gcloud auth application-default login
```
**What it does:** A SECOND login. This is the confusing one.

- The first login (`auth login`) is for *you* typing commands at the terminal.
- The second login (`auth application-default login`) is for *programs* that need to call Google APIs on your behalf. Terraform falls in this category.

These two are stored separately. You need both.

**Memory hook:** "You log in twice. Once for your fingers, once for your tools."

```bash
gcloud auth application-default set-quota-project project-ff3d74fb-0980-44c8-a94
```
**What it does:** When programs call Google APIs, those calls cost quota (rate limits). This command says "charge the quota to this project." Fixes a warning you saw.

### 3.2 Project setup

```bash
gcloud projects list
```
Lists all GCP projects you can see. You had one: `My First Project` with ID `project-ff3d74fb-0980-44c8-a94`.

```bash
gcloud config set project project-ff3d74fb-0980-44c8-a94
```
"Default to this project for everything I do." Saves typing `--project=...` on every command.

```bash
gcloud projects update project-ff3d74fb-0980-44c8-a94 --name="Helpdesk Platform"
```
Renames the *display name* of the project. The PROJECT_ID never changes (it's permanent for the project's lifetime). Only the human-readable name changes. Cosmetic only.

### 3.3 Enable APIs

```bash
gcloud services enable compute.googleapis.com
gcloud services enable cloudresourcemanager.googleapis.com
gcloud services enable iam.googleapis.com
```

**Why this is needed:** Every GCP service is OFF by default. This is a cost-safety feature — you can't accidentally rack up bills on something you didn't intend to use. You explicitly opt in.

- `compute.googleapis.com` — Compute Engine. Required for VPC, subnets, firewall rules, VMs.
- `cloudresourcemanager.googleapis.com` — lets Terraform read/manage project-level metadata.
- `iam.googleapis.com` — Identity & Access Management. Required to create service accounts and grant permissions.

```bash
gcloud services list --enabled --filter="..."
```
Verifies the APIs are on. Always confirm.

### 3.4 The service account attempt (we abandoned this)

```bash
gcloud iam service-accounts create terraform-sa --display-name="Terraform Service Account"
gcloud projects add-iam-policy-binding ... --role="roles/editor"
gcloud iam service-accounts keys create ~/.gcp-keys/terraform-sa-key.json --iam-account=...
```

**What this was trying to do:** Create a "robot identity" for Terraform with a downloadable JSON key file. Industry pattern for years.

**Why it failed:** GCP now blocks downloadable key creation by default (org policy `iam.disableServiceAccountKeyCreation`). It's considered insecure — keys leak when developers commit them to git or paste them in Slack.

**What we did instead:** Used Application Default Credentials (ADC) — already configured by `auth application-default login`. Terraform automatically picks ADC. No key file needed. Modern, more secure.

```bash
gcloud iam service-accounts delete terraform-sa@... --quiet
rm -rf ~/.gcp-keys
```
Cleanup of the unused stuff.

**Interview takeaway:** When asked *"how do you authenticate Terraform to GCP?"* you now know three answers:
1. ADC (what we used) — best for local dev
2. JSON key (what we tried) — older pattern, going away
3. Workload Identity Federation — modern, keyless, used in CI/CD pipelines

---

## 4. Every Terraform file, line by line

Your `network/` folder has 4 files (5 if you count the auto-generated `.terraform.lock.hcl`):

### 4.1 `provider.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

**Block 1 — `terraform { ... }`** declares:
- `required_version = ">= 1.5.0"` — "fail loudly if someone runs this with an older Terraform binary."
- `required_providers { google = { ... } }` — "I need the Google provider plugin." HashiCorp maintains a registry of provider plugins; `hashicorp/google` is the official one. `~> 5.0` means "any 5.x version" (semantic versioning shorthand).

**Block 2 — `provider "google" { ... }`** configures the provider:
- `project = var.project_id` — "use whatever value is in the `project_id` variable" (set in `terraform.tfvars`).
- `region = var.region` — same idea for region.

**Mental model:** `provider.tf` is like `import` statements + initial configuration in a Python script. It tells Terraform which library to load and how to set it up.

### 4.2 `variables.tf`

```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP region — asia-south1 = Mumbai (lowest latency for Hyderabad)"
  type        = string
  default     = "asia-south1"
}

variable "vpc_name" {
  description = "Name of the VPC"
  type        = string
  default     = "helpdesk-vpc"
}
```

**Each `variable` block declares an input parameter:**
- `description` — human-readable help
- `type` — what shape the value should be (string, number, bool, list, etc.)
- `default` — what to use if not overridden (optional)

`project_id` has NO default — it must be supplied. `region` and `vpc_name` have defaults that we accept.

**Mental model:** This file is like the function signature `def main(project_id, region="asia-south1", vpc_name="helpdesk-vpc")` in Python.

### 4.3 `terraform.tfvars`

```hcl
project_id = "project-ff3d74fb-0980-44c8-a94"
```

**What it is:** The actual values for the variables declared in `variables.tf`. Terraform automatically reads any file named `terraform.tfvars`.

**Why it's gitignored:** Different environments (dev, qa, prod) would have different project IDs. You don't want the dev project ID hard-coded in the repo. In real teams, this gets injected by CI/CD or stored in a secrets manager.

### 4.4 `main.tf` — the actual infrastructure

This file is 5 logical sections — let's go through each.

#### Section 1: The VPC

```hcl
resource "google_compute_network" "helpdesk_vpc" {
  name                    = var.vpc_name
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
  description             = "Helpdesk platform VPC — hosts Jenkins, tools, and GKE cluster"
}
```

**Reading this line by line:**
- `resource "google_compute_network" "helpdesk_vpc"` — "Create a resource of type `google_compute_network` and refer to it in my code as `helpdesk_vpc`."
- `name = var.vpc_name` — the name that appears in GCP console. Value comes from `variables.tf` → `helpdesk-vpc`.
- `auto_create_subnetworks = false` — "Don't auto-create subnets in every GCP region. I'll define my own." (If true, GCP would create ~30 default subnets immediately.)
- `routing_mode = "REGIONAL"` — VPC routes stay within a region. Cheaper than `GLOBAL`. Fine for our scope.
- `description` — shown in console. Good practice.

**Mental model:** A VPC is your own isolated, virtual data center inside GCP's physical network. Like leasing a fenced compound inside a city.

#### Section 2: Tools subnet

```hcl
resource "google_compute_subnetwork" "tools_subnet" {
  name          = "tools-subnet"
  ip_cidr_range = "10.10.0.0/24"
  region        = var.region
  network       = google_compute_network.helpdesk_vpc.id
  description   = "Subnet for DevOps tooling VMs"
}
```

- `ip_cidr_range = "10.10.0.0/24"` — this subnet has IPs from `10.10.0.0` to `10.10.0.255`. That's 256 IPs (minus 4 GCP reserves = 252 usable). Plenty for Jenkins + tools VMs.
- `network = google_compute_network.helpdesk_vpc.id` — "attach to the VPC we just defined." This is how Terraform chains resources together — the `.id` reference creates a dependency.

**CIDR explained:**
- `10.10.0.0/24` means "the first 24 bits are fixed, last 8 bits vary" — so 2^8 = 256 IPs.
- `/16` would mean "first 16 bits fixed" — 2^16 = 65,536 IPs.
- Smaller `/N` = bigger range.

#### Section 3: Cluster subnet (with secondary ranges for GKE)

```hcl
resource "google_compute_subnetwork" "cluster_subnet" {
  name          = "cluster-subnet"
  ip_cidr_range = "10.20.0.0/20"
  region        = var.region
  network       = google_compute_network.helpdesk_vpc.id

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.30.0.0/16"
  }

  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.40.0.0/20"
  }
}
```

- `10.20.0.0/20` = 4,096 IPs for the GKE node VMs.
- `secondary_ip_range` blocks are **GKE-specific**. Kubernetes needs IPs for:
  - Each **pod** gets its own IP — Google calls this the pod CIDR.
  - Each **service** (load balancer endpoint) gets its own IP — service CIDR.
- We reserve `10.30.0.0/16` (65k IPs) for pods and `10.40.0.0/20` (4k IPs) for services.

**Why we did this now and not in Week 4:** Subnet IP ranges can't be edited after creation. If we don't reserve secondary ranges now, we'd have to delete and recreate the subnet in Week 4. Plan ahead.

#### Section 4: Firewall — allow SSH

```hcl
resource "google_compute_firewall" "allow_ssh" {
  name    = "allow-ssh"
  network = google_compute_network.helpdesk_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]
  description   = "Allow SSH from any IP — tighten in production"
}
```

- `allow { protocol = "tcp"; ports = ["22"] }` — open TCP port 22, which is SSH.
- `source_ranges = ["0.0.0.0/0"]` — from ANY IP on the internet. This is too permissive for production but fine for learning.
- In a real prod setup, you'd restrict to your office IP, or use IAP tunneling, or put VMs in private subnets with a bastion host.

**Default behavior of GCP firewalls:** Everything is DENIED by default. You explicitly allow what you need.

#### Section 5: Firewall — allow internal

```hcl
resource "google_compute_firewall" "allow_internal" {
  name    = "allow-internal"
  network = google_compute_network.helpdesk_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }
  allow {
    protocol = "icmp"
  }

  source_ranges = [
    "10.10.0.0/24",
    "10.20.0.0/20",
    "10.30.0.0/16",
    "10.40.0.0/20"
  ]
  description = "Allow all internal traffic within VPC"
}
```

- Three `allow` blocks for the three protocols VMs typically use to talk to each other.
- `source_ranges` lists the subnets — only traffic from inside those subnets is allowed.
- This rule lets Jenkins master talk to Jenkins slave, Jenkins talk to GKE pods, etc.

#### Section 6: Outputs

```hcl
output "vpc_id" {
  value = google_compute_network.helpdesk_vpc.id
}
output "tools_subnet_name" {
  value = google_compute_subnetwork.tools_subnet.name
}
output "cluster_subnet_name" {
  value = google_compute_subnetwork.cluster_subnet.name
}
```

**What outputs are:** Values printed after `terraform apply` succeeds. Useful when:
1. You want to see what was created (debugging)
2. Other Terraform modules need to reference these values (e.g., Module 2 will need `vpc_id` to put VMs in this VPC)

### 4.5 `.gitignore`

```
**/.terraform/*
*.tfstate
*.tfstate.*
*.tfvars
crash.log
...
```

**Three categories of things we hide from Git:**
1. `**/.terraform/*` — gigabytes of downloaded provider plugins. Regeneratable. Never commit.
2. `*.tfstate*` — Terraform's state file. **THIS IS CRITICAL.** Holds the actual current state of your infrastructure. In real teams, state is stored in a remote backend (GCS, S3) with locking. Never commit state to git.
3. `*.tfvars` — environment-specific values like your project ID.

---

## 5. Every Terraform command, in order

### `terraform init`

```bash
terraform init
```

**What it does internally:**
1. Reads `provider.tf`.
2. Downloads the Google provider plugin (~50 MB) into `.terraform/providers/`.
3. Creates `.terraform.lock.hcl` to pin the plugin version (so future runs are reproducible).
4. Prepares the working directory.

**When to run:** Once per project, and again whenever you change `required_providers` or add a new provider.

### `terraform plan`

```bash
terraform plan
```

**What it does:**
1. Reads all `.tf` files.
2. Reads the current state file (or assumes empty if none).
3. Calls GCP API to see what currently exists.
4. Computes the diff: "what's in my code minus what's in GCP = what to create/change/destroy."
5. Prints the diff. Makes NO changes.

**Why it matters:** Like `git diff` before `git commit`. You review what will change before it changes. Critical for production safety.

**Typical output sections:**
- `+ resource` = will create
- `- resource` = will destroy
- `~ resource` = will modify in place
- `-/+ resource` = will destroy and recreate

Tonight we saw `Plan: 5 to add, 0 to change, 0 to destroy.` All creates, no destructions. Safe.

### `terraform apply`

```bash
terraform apply
```

**What it does:**
1. Runs `plan` again (to be safe).
2. Asks you `Do you want to perform these actions? Enter a value:`.
3. You type `yes`.
4. Calls GCP API to actually create/change/destroy resources.
5. Writes/updates the state file (`terraform.tfstate`).
6. Prints outputs.

**The state file:** Terraform's memory of what it created. It maps your `.tf` resources to real GCP resource IDs. If you delete the state file by accident, Terraform thinks nothing exists and will try to recreate everything — usually causing errors because the resources DO exist in GCP, just unmapped. Treat state with respect.

### Commands you'll use later

- `terraform destroy` — tear down everything Terraform created. Useful at end of dev sessions to avoid charges.
- `terraform fmt` — auto-format your `.tf` files. Run before commit.
- `terraform validate` — check syntax without contacting GCP.
- `terraform output` — show outputs without re-running apply.
- `terraform state list` — see what's in state.
- `terraform import` — pull existing resources into state (rescue mode).

---

## 6. What's actually living in GCP right now

After `terraform apply` succeeded, these 5 things became REAL in your GCP project:

| Resource | Type | Identifier |
|---|---|---|
| `helpdesk-vpc` | VPC Network | Custom mode, regional routing, asia-south1 |
| `tools-subnet` | Subnetwork | 10.10.0.0/24 in asia-south1 |
| `cluster-subnet` | Subnetwork | 10.20.0.0/20 in asia-south1 (plus secondary ranges) |
| `allow-ssh` | Firewall rule | TCP/22 from 0.0.0.0/0 |
| `allow-internal` | Firewall rule | All proto from VPC subnets |

**Cost so far:** ₹0. Network primitives are free. We start paying in Module 2 when VMs are added.

---

## 7. Self-test quiz (answers at the bottom)

Test yourself before re-reading. Try to answer from memory.

1. Why do we need two separate logins (`gcloud auth login` and `gcloud auth application-default login`)?
2. What's the difference between a Project ID and a Project Name?
3. Why are GCP services off by default?
4. Why did the service-account key creation fail, and what did we use instead?
5. What does `auto_create_subnetworks = false` mean and why did we set it?
6. What does the CIDR `10.10.0.0/24` give you in terms of usable IPs?
7. Why does the GKE subnet need secondary IP ranges?
8. What's the difference between `terraform plan` and `terraform apply`?
9. Why is `terraform.tfvars` in `.gitignore`?
10. What would happen if you ran `terraform apply` again right now with no code changes?

---

## 8. Coming next — Module 2 preview

In Module 2 we'll provision **3 Compute Engine VMs** inside your new VPC:
- `jenkins-master` (tools-subnet, runs Jenkins UI)
- `jenkins-slave` (tools-subnet, executes builds)
- `tools-host` (tools-subnet, runs Nexus + SonarQube)

We'll add:
- A new `compute/` Terraform folder
- VM resources with proper machine types, disks, network interfaces
- A firewall rule to expose Jenkins UI port 8080
- Outputs with VM internal + external IPs

Cost reality: 3 small `e2-small` VMs cost ~₹40/day if left on 24/7, ~₹5/day if stopped overnight. Your ₹28,710 credit handles this comfortably.

---

## 9. How to revise this material

1. **Read once now (tonight or tomorrow morning).** Don't memorize. Just absorb structure.
2. **Tomorrow evening:** Take the self-test quiz cold. Don't look at answers. Whatever you can't answer = open the relevant section and re-read.
3. **In 2 days:** Try to rewrite `main.tf` from a blank file, looking only at the resource type names. No copy-paste. Hit errors. Debug. This is the muscle memory.
4. **In a week:** Explain Module 1 out loud to an imaginary interviewer in 3 minutes. Record yourself if possible. If you can't explain it cleanly, the gaps are your next study targets.

The goal isn't to memorize Terraform syntax — it's to develop intuition for the workflow: declare → plan → apply → verify → commit.

---

## 10. Self-test answers

1. `auth login` is for YOU typing commands. `application-default login` is for PROGRAMS (Terraform) calling Google APIs. They're stored separately.
2. Project ID is permanent and globally unique (e.g., `project-ff3d74fb-...`). Project Name is a human-readable display label (e.g., "Helpdesk Platform") — can be renamed anytime.
3. Cost safety. GCP doesn't enable services for you because each service can generate API charges. You opt in explicitly.
4. GCP has a default org policy `iam.disableServiceAccountKeyCreation` that blocks downloadable JSON keys (security best practice, since keys often leak). We used Application Default Credentials (ADC) instead — Terraform auto-uses your user identity from `auth application-default login`.
5. By default a Google VPC auto-creates a subnet in every region (~30 subnets). `false` means "I'll define exactly the subnets I want, in the regions I want." Cleaner, more controlled.
6. `10.10.0.0/24` = first 24 bits fixed → 256 total IPs (10.10.0.0 through 10.10.0.255). Of those, GCP reserves 4 (network, broadcast, gateway, future), leaving **252 usable**.
7. GKE assigns IPs to pods (one per pod) and services (one per service). These need their own IP ranges separate from the node IPs. Secondary ranges = pre-reserved CIDRs for pods and services.
8. `plan` shows what WOULD change without changing anything (read-only). `apply` actually makes the changes after asking for confirmation.
9. It contains environment-specific values (your project ID). Different environments would have different tfvars. Also, in real teams, sensitive values can end up in tfvars (DB passwords, etc.) — never commit those to git.
10. Terraform would call GCP API, see everything in state matches reality, and report `No changes. Your infrastructure matches the configuration.` Idempotent.
