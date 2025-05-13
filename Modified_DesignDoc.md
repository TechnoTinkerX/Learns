# **A Prototype Job Worker Service – Design Document (Revised)**

A secure, extensible job worker system designed for managing the execution lifecycle of compute jobs (start, stop, query, monitor) on Linux systems, with strong emphasis on process isolation, resource control, secure communications, and access governance.

---

## **Table of Contents**

1. [Overview](#1-overview)
2. [System Architecture](#2-system-architecture)
3. [Detailed Design](#3-detailed-design)

   * [3.1 Process Execution Model](#31-process-execution-model)
   * [3.2 Resource Management via Cgroups](#32-resource-management-via-cgroups)
   * [3.3 Streaming Output Logs](#33-streaming-output-logs)
   * [3.4 gRPC API Design](#34-grpc-api-design)
   * [3.5 Authorization and Security](#35-authorization-and-security)
4. [CLI Overview](#4-cli-overview)
5. [Build and Test Strategy](#5-build-and-test-strategy)
6. [Conclusion](#6-conclusion)

---

## **1. Overview**

The Job Worker Service allows clients to submit and manage background jobs securely over a network. The system provides:

* Secure, encrypted communication.
* Fine-grained access control.
* Real-time resource isolation (CPU, memory, disk I/O).
* Output log streaming to clients.
* CLI for local interaction and debugging.

---

## **2. System Architecture**

The architecture consists of the following components:

* **Job Manager (Core Service):** Manages lifecycle of Linux processes and interfaces with OS-level cgroup and filesystem.
* **gRPC Server:** Exposes job control functionality to remote clients securely.
* **Authorization Middleware:** Authenticates and authorizes users based on certificates.
* **Log Collector:** Persists and streams job logs in near real-time.
* **CLI Client:** Allows users to interact with the service from the terminal.

---

## **3. Detailed Design**

### **3.1 Process Execution Model**

Each job is a Linux subprocess launched with isolation and monitoring. Key features of the execution model:

* **Lifecycle:** Jobs are uniquely identified and tracked. Job status includes metadata like PID, exit code, start/stop time, and output log paths.
* **Isolation:** Each job runs in its own cgroup and working directory to avoid collisions.
* **Monitoring:** Periodic polling or OS signals are used to detect job termination and update internal state.
* **State Storage:** A thread-safe, in-memory registry stores active job metadata, periodically flushed to disk for persistence.

### **3.2 Resource Management via Cgroups**

The system uses **Linux control groups (cgroups v1/v2)** to enforce resource usage limits.

* **Configuration Input:** Limits (CPU, memory, disk I/O) are defined in external config files or passed via the API. These can be dynamically reloaded at runtime.
* **Cgroup Setup:**

  * A new cgroup is created per job.
  * Limits are applied by writing to the relevant control files (e.g., `cpu.max`, `memory.max`).
  * The child process is attached to the cgroup before execution.

**Validation:** Config values are validated for format (e.g., `50%` for CPU, `256M` for memory) and capped to prevent overprovisioning.

### **3.3 Streaming Output Logs**

* **Collection:** Job stdout/stderr is redirected to job-specific files (e.g., `job_<ID>.log`).
* **Log Rotation:** Files are rotated if they exceed a size threshold (e.g., 5MB).
* **Streaming Protocol:**

  * The server reads logs incrementally (tail-style) and streams lines over a gRPC stream.
  * The stream is unidirectional; clients receive line-by-line updates in real time.
* **Format Agnostic:** Output may be text, JSON, or binary. The server does not assume text-only content. The MIME type or format can be negotiated via the request metadata if needed.

### **3.4 gRPC API Design**

APIs are cleanly defined via protobuf with consistent naming. Key RPCs include:

| RPC                 | Description                              |
| ------------------- | ---------------------------------------- |
| `Start(JobRequest)` | Start a job with command and arguments   |
| `Stop(JobID)`       | Gracefully terminate the specified job   |
| `Query(JobID)`      | Retrieve job status including exit code  |
| `Stream(JobID)`     | Stream logs as a stream of output chunks |

**Output Assumptions Addressed:**

* The `Stream` endpoint returns a **stream of bytes** or lines without assuming textual content.
* A field for optional encoding/format can be added for richer clients (e.g., `output_format = TEXT | JSON | BINARY`).

**Error Handling:** Standardized error codes for job not found, already stopped, unauthorized access, etc.

### **3.5 Authorization and Security**

#### **Transport Security**

* TLS 1.3 for all client-server communication.
* 4096-bit RSA with X.509-based mutual authentication.

#### **Authentication**

* Each client presents a certificate signed by a trusted internal CA.
* Server verifies identity and extracts metadata from the cert extensions.

#### **Authorization (RBAC)**

* gRPC interceptors enforce **role-based access control**.
* Roles embedded in the certificate define what RPCs a client can invoke:

  * `READER`: Can query or stream jobs.
  * `WRITER`: Can start, stop, query, and stream.

---

## **4. CLI Overview**

A cross-platform CLI tool interacts with the job service securely. Commands include:

* `start <cmd> [args]` – starts a new job.
* `stop <job-id>` – stops a running job.
* `query <job-id>` – fetches status.
* `stream <job-id>` – tails logs in real-time.

**Example:**

```bash
./worker-client start "ls" "/"
./worker-client query abcd-1234
./worker-client stream abcd-1234
```

All CLI commands support:

* `--cert`, `--key`, `--ca` for TLS parameters.
* Optional `--format=json|text` for output parsing.

---

## **5. Build and Test Strategy**

### **Build**

* Modular Makefile with `client` and `server` targets.
* Use of `go mod`, `protoc`, and TLS cert automation for setup.

### **Testing Plan**

| Test Type         | Description                                                  |
| ----------------- | ------------------------------------------------------------ |
| Unit Tests        | Logic for job tracking, cgroup setup, and auth interceptors  |
| Integration Tests | Simulated client-server runs with TLS certs                  |
| Security Tests    | Attempt invalid certs, expired keys, and unauthorized access |
| Load Tests        | Concurrent job starts and log streaming under stress         |

---

## **6. Conclusion**

This design balances simplicity and robustness. By separating process control, resource enforcement, log management, and security enforcement, the system ensures jobs are well-isolated, observable, and safe from misuse. The gRPC interface offers a powerful integration point, while the CLI provides a developer-friendly tool.

Future extensions may include:

* Web-based dashboard.
* Containerized job execution (e.g., via runc).
* Pluggable authorization policies.
