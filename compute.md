# AWS Cloud Practitioner Essentials — Compute

> Reformatted study notes. Blocks marked **📌 Correction / Clarification** fix or sharpen
> something from the raw notes; **💡** blocks are added explanation and exam tips.

---

## 1. Amazon EC2 (Elastic Compute Cloud)

**📌 Correction:** EC2 = **E**lastic **C**ompute **C**loud (not "Elastic Cloud Compute").

Virtual machines in the cloud. You pick the OS (Linux, Windows, macOS), the CPU/memory
size, the storage, and the network placement — then AWS hands you a running server in
minutes instead of the weeks a physical server purchase would take.

EC2 is the classic **IaaS** service: AWS manages the data center, hardware, and
hypervisor; **you** manage the OS, patching, and everything above it.

### Multitenancy

One physical host runs **many** customer VMs on top of a hypervisor.

- Each instance is **isolated** — it cannot see other tenants' memory, disks, or traffic.
- But instances **share** the host's physical resources (CPU sockets, RAM, NIC, disks).

💡 **Why it matters:** multitenancy is what makes EC2 cheap and instantly available.
If you have a compliance or licensing reason to *not* share hardware, that is exactly what
**Dedicated Instances** and **Dedicated Hosts** are for (see §6).

---

## 2. EC2 Instance Families (the 5 categories)

| Family | Optimized for | Typical workloads | Example series |
|---|---|---|---|
| **General purpose** | Balanced CPU / memory / network | Web servers, small–mid databases, code repos, dev/test, "I don't know my profile yet" | M, T |
| **Compute optimized** | High CPU per GB of RAM | Batch processing, gaming servers, HPC, scientific modeling, ad serving | C, Hpc |
| **Memory optimized** | Large RAM | In-memory caches, big relational databases, real-time analytics on large datasets | R, X, U, Z |
| **Accelerated computing** | Hardware accelerators (GPU / FPGA / custom ASIC) | ML training and inference, graphics rendering, video transcoding, floating-point heavy math | P, G, Inf, Trn, F, VT |
| **Storage optimized** | High local disk IOPS / throughput | Data warehouses, distributed file systems, NoSQL, OLTP with heavy local I/O | I, Im, Is, D |

💡 **Exam tip — how the question is usually phrased:**

- "balanced" / "requirements are uncertain" → **General purpose**
- "high performance computing", "batch compute" → **Compute optimized**
- "large in-memory database", "processes large datasets in memory" → **Memory optimized**
- "machine learning", "GPU", "graphics" → **Accelerated computing**
- "high random IOPS to *local* storage" → **Storage optimized**

⚠️ Subtle distinction: *Storage optimized* means fast **local (instance store)** disks.
If the question is about durable, network-attached disks, the answer is **EBS**, not a
storage optimized instance.

---

## 3. Reading an Instance Type Name

Format: `<family><generation><attributes>.<size>`

```
   m 7 g d . xlarge
   │ │ │ │      └── size (nano → 48xlarge, or "metal")
   │ │ │ └───────── d  = local NVMe instance store
   │ │ └─────────── g  = AWS Graviton (Arm) processor
   │ └───────────── 7  = 7th generation
   └─────────────── m  = general purpose family
```

So `m7gd.xlarge` = general purpose, gen 7, Graviton, with local NVMe, xlarge size.

### Family letters

| Letter | Meaning |
|---|---|
| A | Arm-based AWS Graviton processors |
| C | Compute optimized |
| D | Dense storage |
| F | FPGA |
| G | Graphics intensive |
| Hpc | High performance computing |
| I | Storage optimized |
| Im | Storage optimized (1:4 vCPU-to-memory ratio) |
| Is | Storage optimized (1:6 vCPU-to-memory ratio) |
| Inf | AWS Inferentia (ML **inference**) |
| M | General purpose |
| Mac | macOS |
| P | GPU accelerated |
| R | Memory optimized |
| T | Burstable performance |
| Trn | AWS Trainium (ML **training**) |
| U | High memory |
| VT | Video transcoding |
| X | Memory intensive |
| Z | High memory *and* high CPU frequency |

### Attribute suffixes

**Processor / accelerator:**

| Suffix | Meaning |
|---|---|
| `a` | AMD processors |
| `g` | AWS Graviton processors |
| `i` | Intel processors |
| `q` | Qualcomm inference accelerators |
| `b*00` / `gb*00` | NVIDIA Blackwell GPUs |
| `m*` / `m*pro` | Apple chip |

**Capability:**

| Suffix | Meaning |
|---|---|
| `b` | Block storage optimization |
| `d` | Local instance store volumes |
| `e` | Extra storage / memory / GPU memory (depends on family) |
| `flex` | Flex instance |
| `n` | Network and EBS optimized |
| `z` | High CPU frequency |
| `*tb` | Memory size for high-memory instances (3 TiB – 32 TiB) |

💡 **On `T` (burstable):** T instances accrue **CPU credits** while idle and spend them
when busy. Great for spiky, low-average-load workloads (small web servers, dev boxes);
bad for sustained 100% CPU, where credits run out and you get throttled.

**References**

- Compare specs and prices: https://instances.vantage.sh/
- Official naming conventions: https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-type-names.html

---

## 4. Interacting with AWS Services

**Every** interaction with an AWS service is an **API call**. The console, CLI, and SDKs
are all just different front ends onto the same APIs — which is why anything you can
click, you can automate.

| Method | What it is | Best for |
|---|---|---|
| **AWS Management Console** | Web UI with service search and guided workflows | Visual learners, exploration, one-off tasks, viewing dashboards |
| **AWS CLI** | Command-line tool for Windows / macOS / Linux | Automation, shell scripting, repeatable ops, fast bulk work |
| **AWS SDKs** | Language libraries (Python/boto3, JS, Java, Go, .NET, …) | Building AWS calls *into* your application code |

💡 Worth adding: **AWS CloudFormation** and the **CDK** wrap those same APIs as
*infrastructure as code* — you declare the desired state in a template and AWS builds it.

---

## 5. Amazon Machine Images (AMIs)

An **AMI** is the template an EC2 instance is launched from — a pre-baked disk image plus
metadata.

### What is in an AMI

- Operating system and any pre-installed software/agents
- Root volume storage setup (block device mapping)
- Architecture type (x86_64 or Arm64)
- Launch permissions (who is allowed to use this AMI)

**One AMI → many identical instances.** That is the whole point.

### Where AMIs come from

1. **AWS-provided / Quick Start** — standard Amazon Linux, Ubuntu, Windows Server, etc.
2. **Your own custom AMIs** — configure an instance exactly how you want it, then create
   an image from it (a "golden image").
3. **AWS Marketplace** — third-party vendors sell pre-configured, licensed software images.
4. *(Also)* **Community AMIs** — user-shared and unvetted, so treat with caution.

### Why repeatability matters

Every instance from the same AMI starts identical, so dev, test, and prod match. Combine
this with Auto Scaling and new instances launched during a traffic spike are automatically
configured correctly — no manual setup, no config drift.

⚠️ **AMIs are Region-scoped.** An AMI lives in one Region; to use it elsewhere you copy it
to that Region. This trips people up in questions about multi-Region deployment.

---

## 6. EC2 Pricing Models

| Model | Discount | Commitment | Use when |
|---|---|---|---|
| **On-Demand** | Baseline (0%) | None | Spiky or unknown workloads, dev/test, short-lived jobs, new apps |
| **Savings Plans** | Up to **72%** | $/hour spend commitment for **1 or 3 years** | Steady-state usage, but you want flexibility across instance family, size, and Region — also covers **Lambda** and **Fargate** |
| **Reserved Instances** | Up to **72%** | **1 or 3 years** on a specific instance family + Region | Predictable, long-running, unchanging workloads |
| **Spot Instances** | Up to **90%** | None, but AWS can reclaim the instance with a **2-minute warning** | Fault-tolerant / interruptible work: batch jobs, CI, rendering, stateless workers |
| **Dedicated Instances** | Premium price | None | You need hardware not shared with other AWS customers |
| **Dedicated Hosts** | Most expensive | Optional reservation | Strict compliance, or **bring-your-own-license** tied to physical sockets/cores |

**📌 Note on the numbers:** the course quotes "up to 75%" for Reserved Instances; AWS's
current published figure for both RIs and Savings Plans is **up to 72%**. Either may show
up — what matters is the *ranking*: On-Demand < Savings Plans ≈ RI < Spot.

### Dedicated Instances vs. Dedicated Hosts (a common trap)

- **Dedicated Instances** — hardware isolated from other *customers*, but AWS still
  controls placement. You never see the physical server.
- **Dedicated Hosts** — you reserve the **entire physical server** and get visibility into
  its sockets and cores. This is the one you need for socket- or core-based BYOL licensing
  (e.g. certain Windows Server / SQL Server / Oracle licenses).

---

## 7. Scalability vs. Elasticity

| | **Scalability** | **Elasticity** |
|---|---|---|
| Question it answers | *Can* the system grow? | Does it grow **automatically, right now**? |
| Time horizon | Long-term capacity planning | Real-time, minute by minute |
| Direction | Usually adding capacity | Both out **and back in** |
| Payoff | Handles future growth | Cost efficiency — you don't pay for idle capacity |

**Two ways to scale:**

- **Scale up / vertically** — make the existing machine bigger (`t3.medium` →
  `t3.2xlarge`). Simple, but has a ceiling and usually needs a reboot.
- **Scale out / horizontally** — add more machines. This is the cloud-native way: no hard
  ceiling, and it improves fault tolerance too. *(This is what Auto Scaling does.)*

💡 The key insight: elasticity is scalability that is **automatic and bidirectional**.
Scaling **in** (removing instances when demand drops) is the part that saves money, and
the part an on-premises data center cannot do.

---

## 8. Amazon EC2 Auto Scaling

Automatically adjusts the number of EC2 instances in a group to match demand — improving
both **availability** (enough capacity) and **cost** (not too much capacity).

### Scaling approaches

- **Dynamic scaling** — *reactive*: responds to metrics as demand changes right now.
- **Predictive scaling** — *proactive*: uses ML on historical patterns to pre-launch
  capacity ahead of an expected spike.
- *(Also worth knowing)* **Scheduled scaling** — scale on a clock, e.g. up at 08:00 on
  weekdays, down at 20:00.

Dynamic and predictive work together: predictive sets the baseline, dynamic handles the
surprises.

### The role of CloudWatch

**📌 Clarification:** Auto Scaling does not strictly *require* CloudWatch — scheduled and
manual scaling work without it. But **dynamic scaling** is driven by CloudWatch metrics
and alarms (e.g. "average CPU > 70% for 5 minutes"). CloudWatch is the **eyes**; Auto
Scaling is the **hands**.

### The three capacity settings

| Setting | Meaning |
|---|---|
| **Minimum** | The floor. Never fewer than this many instances — protects availability. |
| **Desired** | The target the group tries to maintain right now. This is the number Auto Scaling actively adjusts. |
| **Maximum** | The ceiling. Protects your bill (and blast radius) from runaway scale-out. |

**📌 Correction to a detail in the raw notes:** it is the **desired capacity** that
determines how many instances launch when you create the group. The confusion is
understandable — *if you don't specify a desired capacity, it defaults to the minimum*, so
in that common case the minimum is what launches. But with min 1 / desired 4 / max 10, you
get **4** instances immediately.

**Worked example:** min 2, desired 2, max 10. Traffic spikes → Auto Scaling raises desired
to 8 and launches 6 more. Traffic drops → desired falls back toward 2 and the extras
terminate. You never pay for more than 10, and never serve traffic with fewer than 2.

---

## 9. Elastic Load Balancing (ELB)

A load balancer distributes incoming traffic across multiple targets (EC2 instances,
containers, IP addresses, Lambda functions) so no single target is overwhelmed.

**It is the single point of contact for incoming traffic to an Auto Scaling group.**
Clients hit the load balancer's DNS name and never need to know instance IPs — which is
essential, because with Auto Scaling those instances come and go constantly.

### Benefits

- **Efficient traffic distribution** — spreads load, prevents hotspots, improves utilization.
- **Automatic scaling** — the load balancer itself scales with traffic, and picks up or
  drops backend instances as Auto Scaling changes them.
- **Health checks** — stops sending traffic to unhealthy instances. (Arguably the biggest
  availability win, and oddly under-emphasized in the course text.)
- **Simplified management / decoupling** — front end and back end don't need to know each
  other's topology; AWS handles maintenance, updates, and failover.

### Routing / load balancing algorithms

| Method | How it works |
|---|---|
| **Round robin** | Each request goes to the next server in a cycle — even distribution. |
| **Least connections** | Sends to whichever server has the fewest active connections. |
| **Least response time** | Sends to the fastest-responding server — minimizes latency. |
| **IP hash** | Hashes the client IP so the same client consistently lands on the same server (basic stickiness). |

**📌 Important nuance:** those four are **general load-balancing concepts** — fine for the
exam's conceptual questions, but they are not a menu of ELB settings. What AWS ELB actually
offers:

- **Application Load Balancer (ALB)** — round robin (default) or **least outstanding
  requests**; Layer 7 (HTTP/HTTPS), can route on path, host, or header.
- **Network Load Balancer (NLB)** — **flow hash** on the connection 5-tuple; Layer 4
  (TCP/UDP), ultra-low latency, very high throughput.
- **Gateway Load Balancer (GWLB)** — routes traffic through third-party virtual appliances
  (firewalls, IDS/IPS).
- Session stickiness on ALB uses **cookies**, not IP hash.

---

## 10. Application Architecture: Monolithic vs. Microservices

### Monolithic — tightly coupled

All components (UI, business logic, app server, database logic) are bundled into one
deployable unit with direct dependencies on each other.

- ➖ One component failing can cascade and take the whole application down.
- ➖ You must scale and deploy the entire app together, even if only one part is the bottleneck.
- ➕ Simple to build and reason about at small scale.

### Microservices — loosely coupled

The application is split into independent services that communicate over well-defined
interfaces (APIs, or better: messages and events).

- ➕ One service failing degrades only its own function; the rest keep working.
- ➕ Each service scales, deploys, and can be rewritten independently.
- ➖ More moving parts: networking, observability, and data consistency all get harder.

💡 **The core idea AWS wants you to take away:** *loose coupling*. If component A talks to
component B through a **queue** or an **event bus** instead of calling it directly, then B
being down means messages pile up harmlessly rather than A failing too.

---

## 11. Integration / Decoupling Services

### Amazon EventBridge — event bus

Serverless **event bus**. Sources (your apps, AWS services, third-party SaaS) publish
events; **rules** filter and route them to targets. It handles receiving, filtering,
transforming, and delivering events.

- Pattern: **event-driven** — "something happened", many possible reactions.
- Example: an S3 upload event triggers a Lambda function *and* logs to another system.

### Amazon SQS — message queue

**Queuing** service for reliable component-to-component communication. A producer puts a
message in the queue; a consumer polls it, processes it, then **deletes** it.

- Messages are **not lost** if the consumer is down — they wait in the queue.
- **One message → one consumer** (pull model). Decouples speed: a fast producer and a slow
  consumer can coexist.
- Example: the web tier drops orders into a queue; a worker fleet processes them at its own pace.

### Amazon SNS — publish/subscribe notifications

**Pub/sub** service. A publisher sends a message to an SNS **topic**, and SNS **pushes** it
to **all** subscribers.

- Subscribers can be Lambda functions, SQS queues, HTTP(S) endpoints, email, SMS, or mobile push.
- **One message → many recipients** (push model, fan-out).
- Example: an "order placed" topic simultaneously notifies billing, shipping, and the customer.

### Quick comparison

| | **SQS** | **SNS** | **EventBridge** |
|---|---|---|---|
| Pattern | Queue (point-to-point) | Pub/sub (fan-out) | Event bus (routing rules) |
| Delivery | Consumer **pulls** | SNS **pushes** | EventBridge **pushes** |
| Recipients | One consumer per message | All subscribers | Targets matching a rule |
| Buffers work? | **Yes** — messages persist until deleted | No | No |
| Reach for it when | You need a durable work backlog | You need to notify many things at once | You need content-based routing, or AWS/SaaS event sources |

💡 **Common combo (fan-out pattern):** SNS topic → several SQS queues → a worker fleet per
queue. You get both broadcast *and* durable buffering.

---

## 12. Quick Recap

- **EC2** = VMs you manage; AWS manages the hardware. **Multitenancy** is what makes it cheap.
- **5 instance families** map to 5 resource bottlenecks: balanced, CPU, memory,
  accelerator, local disk.
- **AMIs** are launch templates — repeatable, identical instances; **Region-scoped**.
- **Pricing spectrum:** On-Demand (flexible, priciest) → Savings Plans / RIs (commit, save
  ~72%) → Spot (interruptible, save ~90%). Dedicated Hosts/Instances = no shared hardware.
- **Scalability** = *can* grow. **Elasticity** = grows **and shrinks, automatically**.
- **Auto Scaling** (min / desired / max) + **CloudWatch** metrics = a right-sized fleet.
- **ELB** = single entry point, spreads traffic, health-checks targets, hides churning
  instance IPs.
- **Loose coupling** (SQS / SNS / EventBridge) is what turns a fragile monolith into a
  resilient system.

---

## 13. Still to Come in the Compute Module

Your notes stop at SNS. The Compute domain also covers, and the exam will ask about:

- **AWS Lambda** — serverless functions; no servers to manage, pay per request and
  duration, event-triggered.
- **Containers** — **Amazon ECS** and **Amazon EKS** (Kubernetes) for orchestration,
  **AWS Fargate** as the serverless compute engine underneath either one, and **Amazon ECR**
  for image storage.
- **Choosing between them** — EC2 (full control, you patch the OS) vs. containers
  (portable, dense) vs. Lambda (no infrastructure, but a 15-minute max execution time).
