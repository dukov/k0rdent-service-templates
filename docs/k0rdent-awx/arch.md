# Architecture

## Overview

This document describes the architecture of the AWX service for K0rdent.

## Components
- K0rdent Cluster
- AWX Service
- Ansible Collection for K0rdent
  - Dynamic Inventory for K0rdent Child Clusters
  - Ansible playbooks and roles for AWX Service Configuration
  - Ansible roles for K0rdent Child Clusters Configuration
- Configuration Repository with user defined configurations for nodes in the inventory

### High Level Diagram

![High Level Diagram](./imgs/high-level-arch.svg)

### Service Diagram

![Service Diagram](./imgs/service-diagram.svg)

## Deployment Flow
```mermaid
sequenceDiagram
    K0rdent User->>K0rdent: Deploy K0rdetn AWX service
    K0rdent->>Mothership K8s: Create Configuration ConfigMap
    Mothership K8s->>K0rdent: Created
    K0rdent->>Mothership K8s: Create Job and mount ConfigMap
    Mothership K8s->>K0rdent: Created
    Mothership K8s->>AWX Configuration Job: Run Container
    AWX Configuration Job->>Ansible CLI: Install requirements
    Ansible CLI->>K0rdent Collection Repo: Pull collection
    K0rdent Collection Repo->>Ansible CLI: Collection contents
    Ansible CLI->>AWX Configuration Job: Success
    AWX Configuration Job->>Ansible CLI: Run AWX config playbook with vars from ConfigMap
    Ansible CLI->>AWX Service: Creat Enttities
    AWX Service->>Ansible CLI: OK
    Ansible CLI->>AWX Configuration Job: Success
    AWX Configuration Job->>Mothership K8s: Job Finished
```