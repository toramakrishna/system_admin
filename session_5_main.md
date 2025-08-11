# Session 5: Network Information System (NIS) and Network File System (NFS)

## System Administration Training 2025-26
### Artificial Intelligence Lab, SCIS

---

## Table of Contents

1. [Network Information System (NIS)](./session_5_nis.md)
2. [Network File System (NFS)](./session_5_nfs.md)
3. [Integration and Best Practices](./session_5_integration.md)
4. [Security Considerations](./session_5_security.md)
5. [Troubleshooting Guide](./session_5_troubleshooting.md)
6. [Practical Lab Exercises](./session_5_lab_exercises.md)
7. [Performance Optimization](./session_5_performance.md)

---

## Session Overview

This session covers essential network services for centralized user management and file sharing in the AI Lab environment:

### **Network Information System (NIS)**
- Centralized user and group management
- Password synchronization across multiple systems
- Host and service information distribution
- Network-wide configuration consistency

### **Network File System (NFS)**
- Distributed file system for sharing directories
- Transparent file access across network
- Centralized storage management
- Home directory sharing

---

## Learning Objectives

By the end of this session, participants will be able to:

1. **NIS Implementation**:
   - Set up NIS master and slave servers
   - Configure NIS clients for user authentication
   - Manage NIS maps and databases
   - Implement secure NIS practices

2. **NFS Deployment**:
   - Configure NFS servers and exports
   - Mount NFS shares on client systems
   - Implement NFS security and access controls
   - Optimize NFS performance

3. **Integration Skills**:
   - Combine NIS and NFS for complete solution
   - Implement high availability configurations
   - Monitor and troubleshoot network services
   - Apply security best practices

4. **AI Lab Specific**:
   - Design user environment for research workflows
   - Implement shared storage for datasets and models
   - Configure development environment consistency
   - Enable collaborative computing resources

---

## Prerequisites

- Completion of Sessions 1-4
- Understanding of Linux user/group management
- Network configuration knowledge
- Basic security concepts
- Familiarity with system services (systemctl)

---

## Lab Environment Architecture

```
AI Lab Network Architecture
==========================

                    [NIS Master Server]
                           |
        ┌──────────────────┼──────────────────┐
        |                  |                  |
   [Workstation 1]   [Workstation 2]   [Workstation 3]
        |                  |                  |
        └──────────────────┼──────────────────┘
                           |
                    [NFS File Server]
                    /home, /projects,
                    /datasets, /models
```

### Typical AI Lab Requirements

- **Centralized User Management**: Researchers need consistent access across all systems
- **Shared Storage**: Large datasets, models, and code repositories
- **Home Directory Roaming**: Users access same environment from any workstation
- **Resource Sharing**: GPU servers, compute clusters, storage arrays
- **Backup and Archival**: Centralized backup of research data

---

## Session Structure

### Part 1: NIS (Network Information System)
- **Theory**: Understanding directory services and centralized authentication
- **Setup**: Installing and configuring NIS master/slave servers
- **Client Configuration**: Connecting workstations to NIS domain
- **Management**: User and group administration via NIS
- **Security**: Implementing secure NIS practices

### Part 2: NFS (Network File System)
- **Theory**: Distributed file systems and remote mounting
- **Server Setup**: Configuring NFS exports and permissions
- **Client Setup**: Mounting NFS shares and automount
- **Performance**: Optimizing NFS for different workloads
- **Security**: NFS security models and Kerberos integration

### Part 3: Integration and Advanced Topics
- **Combined Deployment**: NIS + NFS for complete solution
- **High Availability**: Redundancy and failover mechanisms
- **Monitoring**: Performance metrics and health checking
- **Troubleshooting**: Common issues and diagnostic tools

---

## Real-world AI Lab Scenarios

### Scenario 1: Research Group Setup
- 20 researchers, 10 workstations, shared GPU cluster
- Each researcher needs access to personal and shared datasets
- Code repositories and model checkpoints need to be accessible
- Consistent Python/R environments across all systems

### Scenario 2: Course Environment
- 50 students, computer lab with 25 workstations
- Students need roaming profiles and shared course materials
- Assignment submission and grading workflows
- Restricted access to certain resources

### Scenario 3: High-Performance Computing
- Compute cluster with job scheduler integration
- Large-scale data processing workflows
- Model training with checkpoint sharing
- Results storage and analysis pipelines

---

## Getting Started

1. **Verify Network Connectivity**: Ensure all systems can communicate
2. **DNS Configuration**: Proper hostname resolution is crucial
3. **Time Synchronization**: NTP setup for consistent timestamps
4. **Firewall Rules**: Configure appropriate port access
5. **User Planning**: Design user/group structure before implementation

---

## Best Practices for AI Lab

1. **Security First**: Always implement authentication and encryption
2. **Performance Optimization**: Tune for large file workloads
3. **Backup Strategy**: Multiple copies of critical research data
4. **Documentation**: Maintain detailed configuration records
5. **Monitoring**: Proactive monitoring of all services
6. **User Training**: Educate users on proper usage and security

Navigate to the individual topic files to begin your learning journey!
