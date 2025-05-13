# **A Prototype Job Worker Service**

A simple and secure solution for managing (Start, Stop & Query) and running jobs, with strong control over resources, secure communication, and an easy-to-use interface. This design emphasizes user-friendly resource management, where resource constraints are set via files and dynamically read during job start.

---

## **Table of Contents**

1. **[Overview](#overview)**  
2. **[Core Components](#core-components)**  
   - [Process Management](#1-process-management)  
   - [Resource Management](#2-resource-management)  
   - [gRPC API and Protobuf Definitions](#3.grpc-api-and-protobuf-definitions)  
   - [Command-Line Interface (CLI)](#4-command-line-interface-cli)  
3. **[Security Features](#security-features)**  
4. **[Build and Testing](#build-testing)**  
5. **[Conclusion](#conclusion)**

---

## **1. Overview**

The Prototype Job Worker Service provides an efficient framework for scheduling, monitoring, and managing Linux jobs. It incorporates the following features:  
- Dynamic resource constraints read from user-configured files.  
- Secure communication with **TLS encryption**.  
- A secure **gRPC API** for remote job management.
- An intuitive **CLI** for easy job control.

---

## **2. Core Components**

### **2.1 Process Management**

The core library handles job lifecycle operations and ensures resource restrictions using internal Linux cgroups system commands.

**Responsibilities**:  
- Dynamically read resource limits from files during job execution.  
- Manage job lifecycle: start, stop, and query jobs.
- Stream and store stdout/stderr logs to disk. 

**Structures**:  
```go
type Command struct {
    Name    string   // Name of the Program name
    Args    []string // Arguments for the program
    JobID   string   // Unique job identifier
}

type Job struct {
    ID       string         // Unique job identifier, Job ID
    Process  *os.Process    // Process handle
    Status   string         // Job status (Running, Stopped, etc.)
    CgroupID string         // Associated cgroup for resource control
    Logs     []string       // Collected logs (stdout/stderr)
}
```

**Key Functions**:  
```go
func StartJob(command Command) (*Job, error) {
    // Read resource configurations dynamically from files
    resourceConfig, err := ReadResourceConfigFromFiles()
    if err != nil {
        return nil, err
    }

    // Initialize cgroup with resource limits
    cgroupPath, err := InitializeCgroup(command.JobID, resourceConfig)
    if err != nil {
        return nil, err
    }

    // Start the process
    cmd := exec.Command(command.Name, command.Args...)
    if err := cmd.Start(); err != nil {
        return nil, err
    }

    // Attach process to cgroup
    AttachProcessToCgroup(cgroupPath, cmd.Process.Pid)

    // Return job metadata
    return &Job{ID: command.JobID, Process: cmd.Process, Status: "Running", CgroupID: cgroupPath}, nil
}
```

---

### **2.2 Resource Management**

Resources are controlled using **Linux cgroups**, with limits set via files.  

**Workflow**:  
1. **Resource Files**: Each resource (CPU, memory, disk I/O) is represented by a file (`cpu_limit.txt`, `memory_limit.txt`, etc.).  
2. **Dynamic Read**: Resource limits are dynamically read from these files when a job starts.  

**Key Functions**:  
```go
func ReadResourceConfigFromFiles() (ResourceConfig, error) {
    cpu, err := os.ReadFile("/path/to/cpu_limit.txt")
    if err != nil {
        return ResourceConfig{}, fmt.Errorf("failed to read CPU limit: %v", err)
    }

    memory, err := os.ReadFile("/path/to/memory_limit.txt")
    if err != nil {
        return ResourceConfig{}, fmt.Errorf("failed to read memory limit: %v", err)
    }

    diskIO, err := os.ReadFile("/path/to/disk_io_limit.txt")
    if err != nil {
        return ResourceConfig{}, fmt.Errorf("failed to read disk I/O limit: %v", err)
    }

    return ResourceConfig{
        CPUQuota:    string(cpu),
        MemoryLimit: string(memory),
        DiskIO:      string(diskIO),
    }, nil
}
```

**Example Resource Files**:  
- `/path/to/cpu_limit.txt`: `50%`  
- `/path/to/memory_limit.txt`: `256M`  
- `/path/to/disk_io_limit.txt`: `10MB/s`

**Cgroup Initialization**:  
```go
func InitializeCgroup(jobID string, resourceConfig ResourceConfig) (string, error) {
    cgroupPath := fmt.Sprintf("/sys/fs/cgroup/%s", jobID)
    os.MkdirAll(cgroupPath, 0755)

    // Apply CPU limit
    os.WriteFile(filepath.Join(cgroupPath, "cpu.cfs_quota_us"), []byte(resourceConfig.CPUQuota), 0644)

    // Apply memory limit
    os.WriteFile(filepath.Join(cgroupPath, "memory.limit_in_bytes"), []byte(resourceConfig.MemoryLimit), 0644)

    // Apply disk I/O limit (if applicable)
    if resourceConfig.DiskIO != "" {
        os.WriteFile(filepath.Join(cgroupPath, "blkio.throttle.write_bps_device"), []byte(resourceConfig.DiskIO), 0644)
    }

    return cgroupPath, nil
}
```

---

### **2.3 gRPC API and Protobuf Definitions**

The gRPC API enables remote job management. It includes methods to Start, Stop, Query, and Stream logs for jobs.

#### **Proto Definitions**:  
```protobuf
message StartRequest {
  string name = 1;           // Name of the program to execute (e.g., "myProgram")
  repeated string args = 2;  // List of arguments to pass to the program (e.g., ["arg1", "arg2"])
}

message StartResponse {
  string jobID = 1;
}

message StopRequest {
  string jobID = 1;
}

message StopResponse {
}

message QueryRequest {
  string jobID = 1;
}

message QueryResponse {
  int32 pid = 1;
  int32 exitCode = 2;
  bool exited = 3;
}

message StreamRequest {
  string jobID = 1;
}

message StreamResponse {
  string output = 1;
}

// JobService defines the service with methods to start, stop, query, and stream jobs.
service JobService {
  // Start starts a job with the given StartRequest and returns a StartResponse.
  rpc Start(StartRequest) returns (StartResponse);

  // Stop stops a running job, given the StopRequest, and returns a StopResponse.
  rpc Stop(StopRequest) returns (StopResponse);

  // Query queries the status of a job, given a QueryRequest, and returns a QueryResponse.
  rpc Query(QueryRequest) returns (QueryResponse);

  // Stream streams the output of a job, given a StreamRequest, and returns a stream of StreamResponses.
  rpc Stream(StreamRequest) returns (stream StreamResponse);
}
```


#### **Server-Side Implementation**:  
```go
func (s *Server) Start(ctx context.Context, req *StartRequest) (*StartResponse, error) {
    command := Command{Name: req.Name, Args: req.Args, JobID: generateJobID()}
    job, err := StartJob(command)
    if err != nil {
        return nil, err
    }
    return &StartResponse{JobID: job.ID}, nil
}
```

The server-side worker library processes client requests sent via gRPC, interacts with the Linux operating system through system calls, and runs commands. It is in charge of managing Linux processes by performing start, stop, and query operations, and handling errors that may arise during execution. The worker also stores process outputs (stdout/stderr) in memory. For job management, the worker performs key tasks like starting, stopping, and querying jobs (which represent Linux processes). Mutex locks are implemented to ensure thread safety and avoid race conditions or deadlocks.

### **Start Request**  
- Generates a unique Job ID using `uuid`.
- Executes the specified command via the Go `exec` system call.
- Creates a log file to capture process output (stdout/stderr).
- Stores job metadata, such as PID and status, in memory.
- Sets up resource control using cgroup directories (CPU, memory, IO).

### **Stop Request**  
- Accepts a Job ID to identify the job to stop.
- Retrieves the PID associated with the job.
- Sends a `SIGTERM` signal to terminate the process.
- Returns an error if the job is already finished or doesn’t exist.

### **Query Request**  
- Accepts a Job ID to query the process details.
- Retrieves and displays information such as PID, exit code, and current status.
- The process status is periodically updated in the background.
- Returns an error if the job doesn’t exist.

### **Stream Request**  
- Accepts a Job ID to retrieve the associated log file.
- Streams the contents of the log file (stdout/stderr) related to the job.
- Uses log rotation as necessary to manage log files.
- Returns an error if the job does not exist or if the log file is unavailable.

### Key Design Decisions  
- **Concurrency**: Mutex locks ensure thread safety across client requests, avoiding race conditions and deadlocks.  
- **Resource Control**: Implements CPU, memory, and I/O limits via cgroups.  
- **Error Handling**: Clear and consistent error responses for invalid or completed jobs.  

---

### **2.4 Command-Line Interface (CLI)**

The CLI allows users to interact with the system, supporting resource control parameters during job creation.
1. **Start**
```bash
./bin/worker-client start ls /
# Output: Job ID: a3c1e8b4-90f3-4c6a-ae2f-58d3f827b219 Started
```

2. **Stop**
```bash
./bin/worker-client stop a3c1e8b4-90f3-4c6a-ae2f-58d3f827b219
# Output: 
```

3. **Query** 
```bash
./bin/worker-client query a3c1e8b4-90f3-4c6a-ae2f-58d3f827b219
# Output: Pid: 896581 Exit code: 0 Exited: true
```

4. **Stream** 
```bash
./bin/worker-client stream a3c1e8b4-90f3-4c6a-ae2f-58d3f827b219
# Output: File/Directory list from root
```
---
## **3. Security Features**
**Security:**  
To ensure secure communication between the client and server, we use **Transport Layer Security (TLS 1.3)**, which protects the privacy and integrity of the data. Both the client and server use a self-signed certificate authority (CA). For added security, we use a **4096-bit RSA key**, offering stronger encryption than the usual 2048-bit key, which is ideal for protecting data over a long period.

**Mutual Authentication:**  
We implement **mutual authentication** using **X.509 certificates** to verify the identities of both the client and server. These certificates are signed with the **sha256WithRSAEncryption** algorithm, and the **RSA Public-Key** is **4096 bits** for enhanced security. If the signature is not valid, the connection is rejected, ensuring trust between the client and server.

**Role-Based Access Control (RBAC):**  
Roles are assigned to users and included in the client certificate as **X.509 extensions**. The server then checks these roles using **gRPC interceptors** to determine what actions a user is authorized to perform. The roles include **reader** and **writer**. Writers can perform all available operations like starting, stopping, querying, and streaming, while readers can perform query and stream operations.

---
## **4. Build and Testing**

- **Dynamic Resource Testing**: Validate that resource limits take effect by modifying files before starting jobs.  
- **Security Testing**: Verify TLS encryption and mutual authentication.  

### **Build & Test Commands**: 
# Build and run the gRPC server
```bash
$ make server
go build -o ./bin/worker-server app/server/main.go
```
```
$ ./bin/worker-server
```
# Build and run the gRPC client
```bash
$ make client
go build -o ./bin/worker-client app/client/main.go
```

---

## **5. Conclusion**
This design leverages gRPC to efficiently manage job execution, using protobuf for query transmission instead of the traditional JSON format. By leveraging the Linux cgroups system commands for setting the resources limits, the system ensures robust resource control, limiting CPU, memory, and disk I/O for each job. It also includes secure communication via TLS 1.3 and mutual authentication with X.509 certificates. Additionally, gRPC interceptors are utilized to enforce role-based access control (RBAC), ensuring only authorized users can access specific API methods.

This comprehensive approach makes the system secure, efficient, and reliable for executing jobs with real-time monitoring and strong resource management.
