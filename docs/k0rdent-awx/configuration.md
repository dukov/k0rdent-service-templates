# Configration 

## Overview

This document describes the configuration of the AWX service for K0rdent.

## Quick Start
1. Deploy K0rdent accordgin to official documentation [here](https://docs.k0rdent.io/latest/quickstarts/quickstart-1-mgmt-node-and-cluster/).
1. Create Configuration Repository with user defined configurations for nodes in the inventory (see example [repository](https://github.com/dukov/k0redent-ansible-test)).
   1. Create `k0rdent_nodes.yaml` file in the repository.
      ```yaml
      plugin: k0rdent.core.k8s_nodes
      clusters:
        - type: adopted
          name: mothership
          namespace: kcm-system
          kubeconfig_secret:
            name: adopted-cluster-kubeconfig
            namespace: kcm-system
            key: value
          host_groups:
            - key: node-role.kubernetes.io/worker
              group_prefix: worker_
            - key: node-role.kubernetes.io/control-plane
              group_prefix: controller_
      ansible_subnet: 10.50.50.0/24
      host_groups: []
      ```
   1. Create `requirements.yml` file in the repository.
      ```yaml
      collections:
        - name: https://github.com/dukov/ansible-collections-k0rdent.git
          type: git
          version: main
      ```
   1. Create `production.yml` file in the repository.
      ```yaml
      - hosts: all
        tasks:
          - name: Hello
            ansible.builtin.debug:
              msg: "Hello World!"
      ```
1. Create AWX operator ServiceTemplate using [Kordent Generic Service Template](https://catalog.k0rdent.io/latest/kgst/).
   ```bash
   helm upgrade --install awx-operator oci://ghcr.io/k0rdent/catalog/charts/kgst \
     --set "repo.spec.url=https://ansible-community.github.io/awx-operator-helm/" \
     --set "repo.name=awx-operator" \
     --set "chart=awx-operator:3.2.0" \
     --set "repo.spec.type=default" \
     -n kcm-system
   ```
1. Create k0rdent AWX ServiceTemplate using [Kordent Generic Service Template](https://catalog.k0rdent.io/latest/kgst/).
   ```bash
   helm upgrade --install k0rdent-awx oci://ghcr.io/k0rdent/catalog/charts/kgst \
     --set "repo.spec.url=https://dukov.github.io/k0rdent-service-templates/" \
     --set "repo.name=k0rdent-awx" \
     --set "chart=k0rdent-awx:0.1.2" \
     --set "repo.spec.type=default" \
     -n kcm-system
   ```
1. Create MultiClusterService manifest
   ```yaml filename=mgmt-mcs.yaml
   apiVersion: k0rdent.mirantis.com/v1beta1
   kind: MultiClusterService
   metadata:
     name: mgmt-mcs
     namespace: kcm-system
   spec:
     clusterSelector:
       matchLabels:
         k0rdent.mirantis.com/management-cluster: "true"
         sveltos-agent: present
     serviceSpec:
       provider:
         selfManagement: true
       services: 
         - template: awx-operator-3-2-0
           name: awx-operator
           namespace: awx
           values: |-
             AWX:
               enabled: true
               name: awx
               spec:
                 postgres_data_volume_init: true
                 service_type: LoadBalancer
                 create_preload_data: false
         - template: k0rdent-awx-0-1-2
           name: awx-config
           namespace: awx
           dependsOn: 
             - name: awx-operator
               namespace: awx
           values: |-
             awxServiceUrl: http://awx-service
             awxAdminPasswordSecretName: awx-admin-password
             awxAdminPasswordSecretKey: password
             awxConfig:
               sshKeySecretName: ansible-ssh-key
               awxOrganizations:
                 - name: "Default"
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
   and apply it.
   ```bash
   kubectl apply -f mgmt-mcs.yaml -n kcm-system
   ```
1. Wait for the AWX Service to be deployed and configured.
   ```bash
   kubectl get pods -n awx
   ```
   should show the following pods:
   ```
   NAME                                               READY   STATUS      RESTARTS   AGE
   awx-migration-24.6.1-9tw2x                         0/1     Completed   0          5h52m
   awx-operator-controller-manager-645d4f8c74-s7kg8   2/2     Running     0          5h53m
   awx-postgres-15-0                                  1/1     Running     0          5h52m
   awx-task-69856d5df-d95kv                           4/4     Running     0          5h52m
   awx-web-6b69dd57cd-bfthz                           3/3     Running     0          5h52m
   ```
1. Grab AWX admin password from the Secret and AWX Web UI IP from Service.
   ```bash
   kubectl get secret awx-admin-password -n awx -o jsonpath='{.data.password}' | base64 -d
   kubectl -n awx get svc awx-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
   ```
   and use it to login to the AWX UI.
   ```
   http://<awx-service-ip>
   ```
1. You can see project `k0rdent` and inventory `k0rdent` in the AWX UI.


## Configuration

### AWX Service Configuration

AWX Service is configured using Ansible playbook defined in the [Ansible collection for K0rdent](https://github.com/dukov/ansible-collections-k0rdent).
This playbook is run as a Kubernetes Job after the AWX Service is deployed. It is responsible for configuring AWX organizations, projects, inventories, and instance groups. Kubernetes Job is created by k0rdent-awx service template and executed each time ServerTemplate configuration is updated. Configuration is stored in a ConfigMap and passed to the Job. Follow the [Chart README](../charts/k0rdent-awx/README.md) for detailed configuration options and [K0rdent Ansible Collection documentation](https://github.com/dukov/ansible-collections-k0rdent/blob/main/README.md#roles) for specific parameters description.

### K0rdent Dynamic Inventory

K0rdent Dynamic Inventory is used to discover nodes in the K0rdent Child Clusters.
It is defined in the [Ansible collection for K0rdent](https://github.com/dukov/ansible-collections-k0rdent). Dynamic inventory configuration should be stored in Configuration Repository. 

### Ansible roles for K0rdent Child Clusters Configuration
TBD