# Docker Fundamentals

## 1. Why Docker?

* Traditional Deployment Challenges
* "Works on My Machine" Problem
* Why the Industry Needed Containers

## 2. Linux Process Fundamentals

* Program vs Process
* Process IDs (PID)
* Parent and Child Processes
* Why Containers Are Actually Processes

## 3. Need for Isolation

* Process Visibility Challenges
* Resource Conflicts
* Security Concerns
* Dependency Management Problems

## 4. Containers vs Virtual Machines

* Virtual Machine Architecture
* Container Architecture
* Key Differences
* Why Containers Are Lightweight

## 5. Linux Technologies Behind Docker

* Namespaces
* cgroups
* OverlayFS / Union Filesystems

## 6. How Docker Creates Containers

* What Happens During `docker run`
* Container Creation Flow
* Container Lifecycle Basics

## 7. Docker Architecture

* Docker CLI
* Docker Daemon (dockerd)
* containerd
* runc
* Linux Kernel

## 8. Container Lifecycle

* Create
* Start
* Stop
* Pause
* Remove

## 9. Important Docker Principles

* Images vs Containers
* PID 1 Concept
* Immutable Images
* Ephemeral Containers
* One Image → Multiple Containers

## 10. Hands-On Demonstrations

* Linux Process Inspection
* Namespaces Overview
* Container Lifecycle Demo
* Docker Architecture Walkthrough

## 11. Summary & Q&A

* End-to-End Container Journey
* Key Takeaways
* Open Discussion

---

### One-Line Session Objective

> By the end of this session, you will understand how Docker leverages Linux kernel features such as Namespaces, cgroups, and OverlayFS to create lightweight, isolated containers and why containers have become the foundation of modern cloud-native applications.
