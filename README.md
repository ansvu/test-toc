# Crucible | OCP Control Plane Replacement

> ❗ _Red Hat does not provide commercial support for the content of this repo. Any assistance is purely on a best-effort basis, as resource permits._

---
```bash
##############################################################################
DISCLAIMER: THE CONTENT OF THIS REPO IS EXPERIMENTAL AND PROVIDED "AS-IS"

THE CONTENT IS PROVIDED AS REFERENCE WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
##############################################################################
```
---

## Purpose
About to show how to use crucible to automate an Openshift Control Plane Replacement and Recover OSD.  
For more high level details of new, modified and re-use roles/configuration from crucible, please click here [High-Level-Detail-Of-OCP-CPLANE-Replacement](https://docs.google.com/presentation/d/17SckWgu6Xh2FHEvee2yAWj0l0O7OcA2Fz1KmLT1CEYM/edit?usp=sharing)

**Note**: Most of the requirements and Dependencies are stated at main crucible github [crucible](https://github.com/redhat-partner-solutions/crucible)
but some small updates when tested the Openshift Control Plane Replacement  

## Software Versions Supported
Crucible targets versions of Python and Ansible that ship with RHEL. At the moment the supported versions are:

- RHEL 8.6
- Python 3.6.8
- Ansible 2.9.27


## OpenShift Versions Tested

- 4.10

## Hardware Tested

- Dell PowerEdge R740

## Assisted Installer versions Tested

- v2.12.1
- v2.11.2
- v2.10.0


### Dependencies

Requires the following to be installed on the deployment host:

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-specific-operating-systems)
- [netaddr](https://github.com/netaddr/netaddr)
- [jmespath](https://github.com/jmespath)
- [skopeo](https://github.com/containers/skopeo)
- [podman](https://github.com/containers/podman/)
- [kubectl + oc](https://docs.openshift.com/container-platform/4.10/cli_reference/openshift_cli/getting-started-cli.html)

## Pre-requisite 
- Openshift Compact Cluster 3+0 or 3+N
- ODF Deployment is optional but it requires to customize the site YAML
- New BM server that needs to replace a failure master node
- Assisted-Installed on Premise
- Place the following prerequisites in this directory (fetched):
  - OpenShift pull secret stored as `pull-secret.txt` (can be downloaded from [here](https://console.redhat.com/openshift/install/metal/installer-provisioned))
  - SSH Public Key stored as `noknom-aicli.pub`
  - SSH Private Key that used by existed OCP Cluster `jumphost_private.pem`
  - etcd_secrets.tls is required when new replacement node does not get etcd secrets created (optional) 
```bash
$ mkir -p fetched/ssh_keys
tree fetched/
fetched/
├── etcd_secrets.tls
├── pull-secret.txt
└── ssh_keys
    ├── jumphost_private.pem
    └── noknom-aicli.pub
```

## Update Crucible Inventory For OCP Control Plane Replacement  

**Note**: The OCP Compact Cluster 3+0 with master-0 to master-2, and master-2 node will be replaced with master-x

- **Inventory Configuration**  
nok_master_replacement_inv_updated_master_2_master_x_v1.yml:

---
```yaml
    ################################
    #   Controlplane Replacement   #
    #     Parameters related       #
    #                              #
    #
    node_to_be_deleted_name: "master-2"
    nok_new_node: "master-x"
    nok_private_key: "{{ fetched_dest }}/ssh_keys/jumphost_private.pem"
    disk_to_zap: "/dev/sdc" #check if existing master disks that used by ceph first to make sure
...
...
    repo_root_path: /root/noknom-aicli/crucible/crucible-ctrlplane-replacement-final # path to repository root

    # Directory in which created/updated artifacts are placed
    fetched_dest: "{{ repo_root_path }}/fetched"

    # Configure possible paths for the pull secret
    # first one found will be used
    # note: paths should be absolute
    pull_secret_lookup_paths:
      - "{{ fetched_dest }}/pull-secret.txt"

    # Configure possible paths for the ssh public key used for debugging
    # first one found will be used
    # note: paths should be absolute
    ssh_public_key_lookup_paths:
      - "{{ fetched_dest }}/ssh_keys/{{ cluster_name }}.pub"

    # Set the base directory to store ssh keys
    ssh_key_dest_base_dir: "{{ fetched_dest }}/ssh_keys"
   
    # The retrieved cluster kubeconfig will be placed on the bastion host at the following location
    kubeconfig_dest_dir: "{{ repo_root_path }}/"
    kubeconfig_dest_filename: "{{ cluster_name }}-kubeconfig"
...
...
    services:
      hosts:
        preinstall_ai:
          ansible_host: 192.168.24.80
          port: 8090
...
...
    nodes:
      vars:
        # Set the login information for any BMCs. Note that these will be SET on the vm_host virtual BMC.
        bmc_user: "root"
        bmc_password: "calvin"
      children:
        masters:
          vars:
            role: master
            vendor: Dell # this example is a virtual control plane
          hosts:
            master-2:
              bmc_address: "192.168.24.158"
              ansible_host: 192.168.24.90
              mac: "B8:CE:F6:56:48:82"
              network_config:
                interfaces:
                  - name: "eno1"
                    mac: "{{ mac }}"
                    addresses:
                      ipv4:
                        - ip: "{{ ansible_host }}"
                          prefix: "25"
                dns_server_ips:
                  - "192.168.24.80"
                routes:
                  - destination: "0.0.0.0/0"
                    address: "192.168.24.1"
                    interface: "eno1"
            master-1:
              bmc_address: "192.168.24.156"
              ansible_host: 192.168.24.88
              mac: "B8:CE:F6:56:3D:A2"
              network_config:
                interfaces:
                  - name: "eno1"
                    mac: "{{ mac }}"
                    addresses:
                      ipv4:
                        - ip: "{{ ansible_host }}"
                          prefix: "25"
                dns_server_ips:
                  - "192.168.24.80"
                routes:
                  - destination: "0.0.0.0/0"
                    address: "192.168.24.1"
                    interface: "eno1"
            master-0:
              bmc_address: "192.168.24.155"
              ansible_host: 192.168.24.87
              mac: "B8:CE:F6:56:B3:CA"
              network_config:
                interfaces:
                  - name: "eno1"
                    mac: "{{ mac }}"
                    addresses:
                      ipv4:
                        - ip: "{{ ansible_host }}"
                          prefix: "25"
                dns_server_ips:
                  - "192.168.24.80"
                routes:
                  - destination: "0.0.0.0/0"
                    address: "192.168.24.1"
                    interface: "eno1"

        day2_master:
          vars:
            role: master
            vendor: Dell
          hosts:
            master-x:
              ansible_host: 192.168.24.91
              bmc_address: "192.168.24.159"
              mac: "B8:CE:F6:56:48:AA"
              network_config:
                interfaces:
                  - name: "eno1"
                    mac: "{{ mac }}"
                    addresses:
                      ipv4:
                        - ip: "{{ ansible_host }}"
                          prefix: "25"
                dns_server_ips:
                  - "192.168.24.80"
                routes:
                  - destination: "0.0.0.0/0"
                    address: "192.168.24.1"
                    interface: "eno1"
```
---

## Customize the Site Configuration For Control Plane Replacement
- **Site Customization For Control Plane Replacement**  
  cat nok_master_replacement.yml  
```yaml
---
- import_playbook: playbooks/nok_power_off_node.yml
- import_playbook: playbooks/nok_saved_parameters.yml
- import_playbook: playbooks/nok_etcd_removal.yml
- import_playbook: playbooks/nok_master_replacement_cleanup.yml
- import_playbook: playbooks/generate_discovery_iso.yml
  when: groups['day2_master'] | default([]) | length > 0
  vars:
    discovery_iso_name: "{{ day2_master_discovery_iso_name }}"
    iso_cluster_id: "{{ add_host_cluster_id }}"

- import_playbook: playbooks/boot_iso.yml
  when: groups['day2_master'] | default([]) | length > 0
  vars:
    discovery_iso_name: "{{ day2_master_discovery_iso_name }}"
    boot_iso_url: "{{ discovery_iso_server }}/{{ day2_master_discovery_iso_name }}"
    boot_iso_hosts: day2_master
- import_playbook: playbooks/nok_replace_day2_nodes.yml
- import_playbook: playbooks/approve_csrs.yml
- import_playbook: playbooks/nok_create_bmh_machine.yml

#--- OSD Recovery
- import_playbook: playbooks/nok_zapdisks.yml
- import_playbook: playbooks/nok_osd_recovery.yml

#sanity checking
- import_playbook: playbooks/nok_cplane_replace_post_checking.yml
```

## Start Run Crucible To Replacement An OCP Control Plane Node 
```bash
ansible-playbook  -i nok_master_replacement_inv_updated_master_2_master_x_v1.yml nok_master_replacement.yml -vvv
```

**Note:** If Compact Cluster with ODF Deployed on Masters, then OSD Recovery Roles will require and one known issue where new OSD will take time to bound and rejoin the existing Ceph Cluster up to 15 minutes. 
  
If OSD recovery within 2-4minutes, the total times to replace a master/control plane node is about 35 Minutes, without OSD recovery it takes 25-30 Minutes. 

## Links
- **Crucible Overview** [Crucible-Overview](https://docs.google.com/presentation/d/1em8btgksVWECht-4yyJs49wj7vuwpunm11NijQ-wq_k/edit#slide=id.gfaff7eae47_5_7)
- **Main Crucible Github** [Crucible-Redhat](https://github.com/redhat-partner-solutions/crucible)
