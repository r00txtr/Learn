# Installing Setting Up an All-in-One and Multi-node OpenStack Environment with Kolla-Ansible on Ubuntu 22.04

OpenStack is a powerful open-source platform for cloud computing, enabling you to manage compute, storage, and networking resources. **Kolla-Ansible** simplifies the deployment of OpenStack by using Ansible playbooks and Docker containers. This guide will walk you through installing OpenStack using Kolla-Ansible on **Ubuntu 22.04.5 LTS (Jammy Jellyfish)**, configured in a virtual environment.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Preparation](#system-preparation)
3. [Network Configuration](#network-configuration)
4. [Installing Dependencies](#installing-dependencies)
5. [Setting Up Python Virtual Environment](#setting-up-python-virtual-environment)
6. [Installing Ansible and Kolla-Ansible](#installing-ansible-and-kolla-ansible)
7. [Configuring Ansible](#configuring-ansible)
8. [Inventory Configuration](#inventory-configuration)
9. [Generating Passwords](#generating-passwords)
10. [Global Configuration](#global-configuration)
11. [Preparing the Storage Backend](#preparing-the-storage-backend)
12. [Bootstrapping the Servers](#bootstrapping-the-servers)
13. [Pre-deployment Checks](#pre-deployment-checks)
14. [Deploying OpenStack](#deploying-openstack)
15. [Post-deployment Steps](#post-deployment-steps)
16. [Accessing OpenStack](#accessing-openstack)
17. [Conclusion](#conclusion)

---

## Prerequisites

- **Ubuntu 22.04.5 LTS (Jammy Jellyfish)** installed on a virtual machine.
- **Hardware Requirements**:
  - **RAM**: 4 GB
  - **Primary Storage**: 60 GB for - the root file system. (`/dev/sda`)
  - **Secondary Storage**: 10 GB - for external storage (used for Cinder LVM). (`/dev/sdb`)
- **Network Interfaces**:
  - `enp0s3`: Host-only Adapter (for internal communications).
  - `enp0s8`: Host-only Adapter (for external communications).
  - `enp0s9`: Bridge Adapter (Wi-Fi Ethernet)

**Note**: Ensure that your virtual machine has the specified hardware configurations and network interfaces set up.

---

## System Preparation

Start by updating your system to ensure all packages are up to date.

```bash
sudo apt update && sudo apt-get full-upgrade -y
```

---

## Network Configuration

Proper network configuration is crucial for OpenStack deployment.

### Check Network Interfaces

Run the following command to list your network interfaces:

```bash
ifconfig
```

**Sample Output**:

```
enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.10.10.110  netmask 255.255.255.0  broadcast 10.10.10.255
        ...

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ...

enp0s9: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.6  netmask 255.255.255.0  broadcast 192.168.1.255
        ...
```

### Configure Netplan

Edit the Netplan configuration file:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

**Content**:

```yaml
network:
    ethernets:
        enp0s3:
            dhcp4: true
        enp0s8:
            dhcp4: true
        enp0s9:
            dhcp4: true
    version: 2
```

**Explanation**:

- **enp0s3**, **enp0s8**, and **enp0s9** are set to obtain IP addresses via DHCP.
- Ensure these interfaces match your actual network interface names.

Apply the Netplan configuration:

```bash
sudo netplan apply
```

### Update /etc/hosts

It's important to have proper hostname resolution. Edit the hosts file:

```bash
sudo nano /etc/hosts
```

Add entries for your hostnames and IP addresses as needed.

---

## Installing Dependencies

Install necessary packages:

```bash
sudo apt-get install python3-dev libffi-dev gcc libssl-dev python3-selinux python3-setuptools python3-venv net-tools -y
```

- **python3-venv**: For creating Python virtual environments.
- **net-tools**: Provides `ifconfig` and other networking tools.

---

## Setting Up Python Virtual Environment

Create a virtual environment to isolate Kolla-Ansible dependencies:

```bash
python3 -m venv kolla-venv
```

Activate the virtual environment:

```bash
source kolla-venv/bin/activate
```

Upgrade `pip`:

```bash
pip install -U pip
```

---

## Installing Ansible and Kolla-Ansible

Install a specific version of Ansible:

```bash
pip install ansible==9.10.0
```

Install Kolla-Ansible:

```bash
pip install kolla-ansible==18.2.0
```

---

## Configuring Ansible

### Create Kolla Configuration Directory

```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

Copy Kolla-Ansible configuration files:

```bash
cp -r kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp kolla-venv/share/kolla-ansible/ansible/inventory/* .
```

### Configure Ansible

Create Ansible configuration directory:

```bash
sudo mkdir -p /etc/ansible
```

Edit the Ansible configuration file:

```bash
sudo nano /etc/ansible/ansible.cfg
```

**Content**:

```ini
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

- **host_key_checking=False**: Disables SSH host key checking.
- **pipelining=True**: Improves Ansible performance.
- **forks=100**: Increases the number of parallel processes.

---

## Inventory Configuration

Kolla-Ansible uses an inventory file to define which hosts will run which services.

You can use the provided `all-in-one` inventory for a single-node deployment or create a `multinode` inventory for multiple nodes.

### Example: Multinode Inventory

Create or edit the `multinode` inventory file:

```bash
nano ~/multinode
```

**Content**:

```
[control]
pod-controller

[network]
pod-controller

[compute]
pod-compute1
pod-compute2

[monitoring]
pod-controller

[storage]
pod-controller
pod-compute1
pod-compute2

[deployment]
localhost ansible_connection=local
```

**Note**: Replace `pod-controller/pod-compute` with your actual hostname/ip.

### Ping Test

Verify connectivity to all hosts:

```bash
ansible -i all-in-one all -m ping
```

**Expected Output**:

```
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## Generating Passwords

Generate passwords for the OpenStack services:

```bash
kolla-genpwd
```

You can view the generated passwords in `/etc/kolla/passwords.yml`.

---

## Global Configuration

Edit the `globals.yml` file to customize your deployment:

```bash
nano /etc/kolla/globals.yml
```

**Essential Configuration Options**:

```yaml
kolla_base_distro: "ubuntu"
openstack_release: "2024.1"
kolla_internal_vip_address: "10.10.10.99"
network_interface: "enp0s3"
neutron_external_interface: "enp0s8"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
```

**Explanation**:

- **kolla_base_distro**: Base Linux distribution for containers.
- **openstack_release**: OpenStack version.
- **kolla_internal_vip_address**: Virtual IP address for internal API endpoints.
- **network_interface**: Interface for management network.
- **neutron_external_interface**: Interface for external network traffic.
- **enable_cinder**: Enables the Cinder service (block storage).
- **enable_cinder_backend_lvm**: Enables LVM backend for Cinder.

---

## Preparing the Storage Backend

Cinder requires a volume group named `cinder-volumes`.

### Identify Secondary Storage

Check available disks:

```bash
sudo fdisk -l
```

**Sample Output**:

```
Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
...
```

### Create Physical Volume

```bash
sudo pvcreate /dev/sdb
```

### Create Volume Group

```bash
sudo vgcreate cinder-volumes /dev/sdb
```

Verify the volume group:

```bash
sudo vgs
```

**Sample Output**:

```
VG             #PV #LV #SN Attr   VSize   VFree
cinder-volumes   1   0   0 wz--n- <10.00g <10.00g
```

---

## Bootstrapping the Servers

Install dependencies:

```bash
kolla-ansible install-deps
```

Bootstrap servers:

```bash
kolla-ansible -i ./all-in-one bootstrap-servers
```

**Note**: If you encounter permission issues with Docker, run:

```bash
sudo chown $USER /var/run/docker.sock
sudo usermod -aG docker $USER
sudo systemctl restart docker
```

---

## Pre-deployment Checks

Run pre-deployment checks to ensure your environment is ready:

```bash
kolla-ansible -i ./all-in-one prechecks
```

---

## Deploying OpenStack

Deploy OpenStack services:

```bash
kolla-ansible -i ./all-in-one deploy
```

This process may take some time as it pulls Docker images and starts services.

---

## Post-deployment Steps

After deployment, perform post-deployment tasks:

```bash
kolla-ansible -i ./all-in-one post-deploy
```

This generates the `admin-openrc.sh` file containing environment variables for accessing OpenStack.

---

## Accessing OpenStack

### Install OpenStack Client

Install the OpenStack CLI client:

```bash
pip install openstackclient
```

### Load OpenStack Credentials

Source the `admin-openrc.sh` file to load environment variables:

```bash
source /etc/kolla/admin-openrc.sh
```

**Note**: If the file is not in `/etc/kolla/`, check the location specified during deployment.

### Verify OpenStack Services

List available services:

```bash
openstack service list
```

**Sample Output**:

```
+----------------------------------+------------+------------+
| ID                               | Name       | Type       |
+----------------------------------+------------+------------+
| 1234567890abcdef1234567890abcdef | keystone   | identity   |
| abcdef1234567890abcdef1234567890 | nova       | compute    |
| ...                              | ...        | ...        |
+----------------------------------+------------+------------+
```
---

## Example of `/etc/kolla/admin-openrc.sh`

Here's an example of the `admin-openrc.sh` file that sets the environment variables for OpenStack CLI:

```bash
cat /etc/kolla/admin-openrc.sh
```

```bash
# Ansible managed

# Clear any old environment that may conflict.
for key in $( set | awk '{FS="="}  /^OS_/ {print $1}' ); do unset $key ; done

export OS_PROJECT_DOMAIN_NAME='Default'
export OS_USER_DOMAIN_NAME='Default'
export OS_PROJECT_NAME='admin'
export OS_TENANT_NAME='admin'
export OS_USERNAME='admin'
export OS_PASSWORD='ewTWB4AMDxdnBcjTZ9nSZQqscwrMJpYKB0rNJbsp'
export OS_AUTH_URL='http://10.10.10.99:5000'
export OS_INTERFACE='internal'
export OS_ENDPOINT_TYPE='internalURL'
export OS_IDENTITY_API_VERSION='3'
export OS_REGION_NAME='RegionOne'
export OS_AUTH_PLUGIN='password'
```

These environment variables are used by the OpenStack CLI to authenticate and communicate with your OpenStack services.

---

## Troubleshooting

### Networking Issues

If instances cannot connect to the network, ensure that youâ€™ve properly configured your network interfaces and the `globals.yml` file:

```bash
network_interface: "enp0s3"
neutron_external_interface: "enp0s8"
```

You may also need to enable IP forwarding and configure IP masquerading:

```bash
sudo sysctl net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.10.10.0/24 ! -d 10.10.10.0/24 -j MASQUERADE
```

---

## Conclusion

In this guide, weâ€™ve walked through the setup and configuration of an OpenStack environment using **Kolla-Ansible** on **Ubuntu 22.04**. You can use this setup for either an all-in-one deployment or a multi-node environment. By following these steps, you can deploy a fully operational OpenStack environment for testing or production purposes.

**Next Steps**:

- Access the Horizon dashboard (if enabled) via a web browser at `http://<kolla_internal_vip_address>/`.
- Explore creating networks, launching instances, and attaching volumes.

**Helpful Tips**:

- Always check logs in `/var/log/kolla/` if you encounter issues.
- Use `kolla-ansible -i ./all-in-one destroy` to clean up your environment if needed.

---

**Screenshots**:

*Due to the text-based nature of this guide, please refer to official OpenStack documentation or online resources for visual references and screenshots related to each step.*

---

By following this guide, you've set up a powerful cloud environment for testing and development. Happy cloud computing!
