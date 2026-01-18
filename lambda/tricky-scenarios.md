
## 🔥 Tricky Scenario-Based AWS Lambda Interview Questions

### **1️⃣ Cold Start Trap**

Your Lambda function has **Provisioned Concurrency enabled**, yet you still see **cold start latency** occasionally.

👉 **Question:**
How is this possible? Explain **two valid reasons**.

**Correct Answer:**

Provisioned Concurrency does **not guarantee zero cold starts**.

Cold starts still happen when:

1. **Incoming concurrency exceeds provisioned concurrency**
2. Traffic hits a **version or alias without PC**
3. **New version deployment** occurs
4. Sudden traffic spikes happen before PC scales

📌 *Provisioned Concurrency is applied per **version + alias**, not per function.*

---

### **2️⃣ Duplicate Processing Nightmare**

Your Lambda is triggered by **SQS Standard Queue** and processes payments.
Customers report **duplicate charges**.

👉 **Question:**
Why can this happen even if Lambda succeeds?
How would you design the system to **guarantee idempotency**?

**Correct Answer:**

Duplicates happen because:

* SQS Standard provides **at-least-once delivery**
* Ordering is **not guaranteed**

Correct mitigation:

* Design **idempotent Lambda logic**
* Use **deduplication keys** (DynamoDB conditional writes)
* Align **visibility timeout > Lambda timeout**
* Use **FIFO queue** if ordering & deduplication are mandatory

📌 *Visibility timeout alone does NOT prevent duplicates.*

---


### **3️⃣ Throttling Without Traffic Spike**

A Lambda function throws **`ThrottlingException`**, but CloudWatch shows **low request rate**.

👉 **Question:**
List **three possible causes** unrelated to traffic volume.

**Correct Answer:**

Throttling can happen even with low request rate due to:

1. **Reserved concurrency limit**
2. **Account-level concurrency limit**
3. Long-running executions holding concurrency
4. Retry storms (EventBridge / async / SQS)
5. Synchronous downstream calls

📌 *Concurrency ≠ Requests per second*

---

### **4️⃣ Invisible Timeout**

A Lambda function times out at **30 seconds**, but CloudWatch logs show it only ran for **22 seconds**.

👉 **Question:**
What could explain this discrepancy?

**Correct Answer:**

Lambda logs only show **application runtime**, not:

* Cold start time
* Init phase
* Network setup

Hence:

* Timeout at 30s
* Logs show ~22s

📌 *Timeout includes init + execution.*

---

### **5️⃣ DynamoDB Streams Lag**

Your Lambda processes **DynamoDB Streams**, but **IteratorAge** keeps increasing.

👉 **Question:**
Why does increasing Lambda timeout **not help**?
What actually fixes this?

**Correct Answer:**

Increasing timeout does NOT help because:

* Streams are processed per **shard**
* Lambda cannot keep up with throughput

Fix:

* Increase **parallelization**
* Increase **shard count**
* Ensure enough concurrency

📌 *IteratorAge = backlog problem, not execution time.*

---

### **6️⃣ /tmp Data Loss Surprise**

You store temporary files in `/tmp` to speed up processing.
Sometimes the files **disappear between invocations**.

👉 **Question:**
Why does this happen even without deployment?
How should the design change?

**Correct Answer:**

* `/tmp` is **ephemeral**
* Execution environment can be **recycled anytime**
* Persistence is **best-effort only**

Correct design:

* Use `/tmp` only as cache
* Store real data in **S3 / EFS / DynamoDB**

---

### **7️⃣ VPC + Internet Failure**

Your Lambda runs inside a **VPC** and suddenly fails to call an external API.

👉 **Question:**
What changed if the code is untouched?
List **all networking components** that must exist for internet access.



## 🔍 **What likely changed (without touching code)**

One or more **networking components** were modified, deleted, or misconfigured:

* NAT Gateway deleted or unavailable
* Route table changed
* Subnet changed (private instead of public)
* Security Group tightened
* Network ACL updated
* Elastic IP released (for NAT)
* VPC endpoint routing conflict
* Lambda ENIs recreated in a subnet without NAT route

📌 *Lambda failures in VPC are almost always networking, not code.*

---

## 🌐 **ALL Networking Components Required for Internet Access**

For a Lambda **inside a VPC** to access the internet, **ALL** of the following must exist and be correctly configured.

---

### **1️⃣ Private Subnet**

* Lambda **must be in a private subnet**
* Private subnet does **NOT** have direct IGW route

Route table:

```
0.0.0.0/0 → NAT Gateway
```

📌 *Lambda cannot use public subnets for internet access.*

---

### **2️⃣ NAT Gateway (or NAT Instance)**

* Required for **outbound internet access**
* Must be:

  * In a **public subnet**
  * Associated with an **Elastic IP**

Without NAT:
❌ No outbound internet

---

### **3️⃣ Public Subnet**

The NAT Gateway lives here.

Route table:

```
0.0.0.0/0 → Internet Gateway
```

---

### **4️⃣ Internet Gateway (IGW)**

* Attached to the VPC
* Enables internet connectivity **for the NAT Gateway**

Without IGW:
❌ NAT cannot reach the internet

---

### **5️⃣ Route Tables**

Two critical routes:

**Private subnet route table**

```
0.0.0.0/0 → NAT Gateway
```

**Public subnet route table**

```
0.0.0.0/0 → Internet Gateway
```

📌 *Wrong route table = instant outage.*

---

### **6️⃣ Security Groups**

* Lambda ENI security group must allow:

  * **Outbound HTTPS (443)** or required port

Example:

```
Outbound: 0.0.0.0/0 → TCP 443
```

📌 *Security groups are stateful.*

---

### **7️⃣ Network ACLs (NACLs)**

* Must allow:

  * Outbound internet traffic
  * **Ephemeral ports (1024–65535)** for return traffic

Common failure:
❌ NACL blocks return traffic

---

### **8️⃣ DNS Resolution Enabled**

VPC settings:

* `enableDnsSupport = true`
* `enableDnsHostnames = true`

Without DNS:
❌ API hostname won’t resolve

---

### **9️⃣ No Conflicting VPC Endpoints**

* Interface or Gateway endpoints may override routing
* Example:

  * Private API resolves internally but external API fails

📌 *Endpoints can silently hijack traffic.*

---

### **🔟 Availability Zone Consistency**

* NAT Gateway must be in **same AZ** as Lambda subnet
  *(best practice)*

Cross-AZ NAT:

* Works
* But failure if AZ NAT is deleted

---

## 🧠 **One-Line Interview Answer**

> “When a Lambda inside a VPC loses internet access without code changes, the root cause is almost always a missing or misconfigured NAT Gateway, route table, security group, NACL, or IGW.”

---

## 🔥 **Ultra-Senior Follow-Up (Bonus Answer)**

**Q:** Why does Lambda NOT get internet by default in a VPC?
**A:** Because Lambda ENIs in a VPC have **private IPs only**, and AWS removes the default internet routing for security.

---

## 🧪 **Quick Debug Checklist (Interview Gold)**

1. Is Lambda in a **private subnet**?
2. Does subnet route to **NAT Gateway**?
3. Is NAT in **public subnet**?
4. Is IGW attached?
5. Is Elastic IP attached to NAT?
6. Are SG + NACL allowing outbound?
7. DNS enabled?
8. Any new VPC endpoints?

---


---

### **8️⃣ Retry Storm**

A Lambda triggered by **EventBridge** calls a flaky downstream service.
Suddenly, downstream systems are overwhelmed.

👉 **Question:**
Why did this happen?
How would you **stop retry storms**?

**Correct Answer:**

Retry storms occur because:

* EventBridge retries **automatically (2 retries)**
* Failures trigger parallel retries
* Downstream systems get overwhelmed

Mitigation:

* Exponential backoff
* Circuit breakers
* Failure destinations
* Decouple with SQS

📌 *Retries amplify failures if uncontrolled.*

---

### **9️⃣ API Gateway + Lambda Latency**

An API Gateway → Lambda setup has **high latency**, but Lambda duration is low.

👉 **Question:**
Where is the latency coming from?
Name **three non-Lambda causes**.

**Correct Answer:**

Latency comes from:

* Lambda authorizers
* VPC Link / NAT
* Mapping templates
* TLS / CloudFront
* Throttling checks

📌 *API Gateway adds latency before Lambda executes.*

---

### **🔟 Concurrency Death Spiral**

A Lambda function calls another Lambda synchronously.

👉 **Question:**
How can this lead to a **concurrency deadlock**?
How do you fix it?

**Correct Answer:**

Deadlock occurs when:

* Lambda A → calls Lambda B synchronously
* Lambda B → calls Lambda A synchronously
* Both wait and consume concurrency

Fix:

* Avoid sync Lambda chaining
* Use async, SQS, or Step Functions

📌 *Never create cyclic synchronous Lambda calls.*

---


### **1️⃣1️⃣ Memory vs Cost Paradox**

Increasing Lambda memory **reduced total cost**, even though price per ms increased.

👉 **Question:**
Explain how this is possible.

**Correct Answer:**

Lambda CPU scales with memory.
Higher memory:

* Executes faster
* Uses fewer GB-seconds
* Reduces total cost

📌 *Cost is GB-seconds, not execution count.*

---

### **1️⃣2️⃣ Payload Limit Gotcha**

Your Lambda fails when invoked via **API Gateway**, but works fine when invoked directly.

👉 **Question:**
What is the likely cause?

**Correct Answer:**

* API Gateway payload limit = **10 MB**
* Direct Lambda invocation allows larger payloads

Solution:

* Upload payload to **S3**
* Pass reference to Lambda

---

### **1️⃣3️⃣ Canary Deployment Gone Wrong**

You use **Lambda aliases** with weighted routing (90/10).
Some users always hit the **old version**.

👉 **Question:**
Why?
How do you ensure fair traffic distribution?

**Correct Answer:**

Uneven traffic occurs because:

* Sticky clients
* Small traffic volume
* Cached DNS / keep-alive connections

Fix:

* Increase traffic volume
* Use weighted alias correctly
* Observe over longer window

---


### **1️⃣4️⃣ DLQ ≠ Reliability**

You configured a **Dead Letter Queue**, but still lose events.

👉 **Question:**
In which scenarios does DLQ **not protect you**?

---

### **1️⃣5️⃣ Hidden Cost Explosion**

Lambda cost suddenly spikes without traffic increase.

👉 **Question:**
What Lambda features commonly cause **silent cost explosions**?

**Correct Answer:**

DLQ does not cover:

* Partial success
* Data corruption
* Sync invocation failures
* Logic bugs

📌 *Idempotency + retries are still required.*

---


### **1️⃣6️⃣ Exactly-Once Myth**

An interviewer says:

> “Lambda guarantees exactly-once execution.”

👉 **Question:**
How do you **politely prove this is false** with examples?

**Correct Answer:**

Lambda **does NOT guarantee exactly-once execution**.

Reasons:

* SQS at-least-once
* Async retries
* Timeouts
* Network failures

Correct approach:

* Idempotent design
* Deduplication keys
* Conditional writes

📌 *Lambda guarantees at-least-once only.*

---
### **1️⃣7️⃣ Logging Kills Performance**

After enabling detailed logging, Lambda latency increases.

👉 **Question:**
Why does logging affect performance?
How do you fix it?

**Correct Answer:**

Logging causes:

* Blocking I/O
* Serialization overhead
* CloudWatch ingestion delay

Fix:

* Reduce log volume
* Use structured logging
* Log only on errors

---

### **1️⃣8️⃣ Reserved vs Provisioned Confusion**

A team enables **Provisioned Concurrency**, but throttling still happens.

👉 **Question:**
What did they misunderstand?

**Correct Answer:**

Rule:

```
Reserved Concurrency ≥ Provisioned Concurrency
```

Provisioned concurrency:

* Consumes reserved concurrency
* Does NOT bypass limits

📌 *Misconfiguration causes throttling.*

---


### **1️⃣9️⃣ Long-Running Workload**

You need a task that runs for **30 minutes**.

👉 **Question:**
Why is Lambda the wrong choice?
What AWS service fits better?

**Correct Answer:**

Lambda max execution = **15 minutes**

Better alternatives:

* AWS Glue
* ECS / Batch
* Step Functions orchestration

---

### **2️⃣0️⃣ Security Trap**

A Lambda can access **Secrets Manager** even though the role seems restrictive.

👉 **Question:**
What IAM misconfiguration could cause this?

**Correct Answer:**

Causes:

* Over-permissive IAM role
* Wildcard policies (`*`)
* Inherited permissions
* Resource-based policies

Fix:

* Least privilege IAM
* Explicit deny
* Scope permissions tightly


---
