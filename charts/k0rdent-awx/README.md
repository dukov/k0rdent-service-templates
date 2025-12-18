# k0rdent AWX Chart

## Overview

This Helm chart deploys a Kubernetes Job that configures AWX (Ansible Automation Platform) using Ansible playbooks. The chart creates the necessary RBAC resources and runs a post-install/post-upgrade hook to set up AWX organizations, projects, inventories, and instance groups. Main use case for the chart is usage as K0rdent Service Template.

## Description

The k0rdent-awx chart automates the configuration of AWX instances by:
- Creating a ServiceAccount with appropriate RBAC permissions
- Running a one-time configuration Job that uses the AWX Execution Environment image
- Configuring AWX resources (organizations, projects, inventories, instance groups) via Ansible playbooks
- Installing Ansible collections from Git repositories

## Prerequisites

- Kubernetes cluster with AWX service running (preferably deployed as K0rdent Service)
- AWX admin password stored in a Kubernetes Secret
- SSH key for Ansible to connect to nodes in the inventory
- Helm 3.x

## Installation

```bash
helm repo add k0rdent-sts https://dukov.github.io/k0rdent-service-templates
helm repo update
helm install <release-name> k0rdent-sts/k0rdent-awx -f values.yaml
```

## Configuration

The following table lists the configurable parameters and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `backoffLimit` | Number of retries before marking the Job as failed | `4` |
| `awxServiceUrl` | URL of the AWX service | `http://awx-service` |
| `awxAdminPasswordSecretName` | Name of the Secret containing AWX admin password | `awx-admin-password` |
| `awxAdminPasswordSecretKey` | Key in the Secret containing the password | `password` |
| `awxConfig.collectionGitUrl` | Git URL for the Ansible collection | `https://github.com/dukov/ansible-collections-k0rdent.git` |
| `awxConfig.collectionGitVersion` | Git version/branch for the collection | `main` |
| `awxConfig.sshKeySecretName` | Name of the Secret containing SSH key (required) | - |
| `awxConfig.awxOrganizations` | List of AWX organizations to create | `[]` |
| `awxConfig.awxProjects` | List of AWX projects to create | `[]` |
| `awxConfig.awxInventories` | List of AWX inventories to create | `[]` |
| `awxConfig.awxInstanceGroups` | List of AWX instance groups to create | `[]` |
| `awxConfig.awxJobTemplates` | List of AWX job templates to create | `[]` |
| `awxConfig.extraCollections` | Additional Ansible collections to install | `[]` |
| `image.repository` | Container image repository | `quay.io/ansible/awx-ee` |
| `image.tag` | Container image tag | `24.6.1` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `serviceAccount.automount` | Automount service account token | `true` |
| `resources` | Resource limits and requests | `{}` |

## Resources Created

This chart creates the following Kubernetes resources:

- **ServiceAccount**: Used by the configuration Job
- **ClusterRole**: Grants permissions to read secrets (for inventory reading)
- **ClusterRoleBinding**: Binds the ServiceAccount to the ClusterRole
- **ConfigMap**: Contains the configuration scripts and Ansible playbook variables
- **Job**: Runs as a Helm post-install/post-upgrade hook to configure AWX

## Usage Example

```yaml
awxConfig:
  sshKeySecretName: ansible-ssh-key
  awxOrganizations:
    - name: "Default"
      description: "K0rdent Organization"
  awxProjects:
    - name: "k0rdent"
      scm_url: "https://github.com/dukov/k0redent-ansible-test"
      organization: "Default"
  awxInventories:
    - name: "k0rdent"
      organization: "Default"
      inventory_sources:
        - name: "k0rdent"
          source_project: "k0rdent"
          source_path: "k0rdent_nodes.yaml"
  awxInstanceGroups:
    - name: "default"
      pod_spec_override:
        spec:
          serviceAccountName: awx-inventory-reader
          automountServiceAccountToken: true
```

## Notes

- The Job runs as a Helm hook (`post-install` and `post-upgrade`) and is automatically deleted after successful completion
- The SSH key secret is required and must be created before installing the chart
- The AWX admin password secret must exist in the same namespace where the chart is installed
- The Job runs with root privileges (fsGroup: 0, runAsUser: 0) by default

