# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this tutorial, all of these machines will run in OpenStack.

## Source your OpenStack RC file.

This operation configures OpenStack clients to interact with your OpenStack project. This file must be sourced in order to perform OpenStack operations.

As user `vagrant` inside the Vagrant machine:

```bash
source /vagrant/openrc.sh
```

## Availability Zones (AZs)

In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [Availability Zone (AZ)][az]. Unforutnately, in some OpenStack clouds, AZs are inconsistently named across the network, storage and compute layers. Therefore, you must select an AZ for each layer.

### Select an AZ for Network.

Example:

```bash
neutron availability-zone-list
# +------+----------+-----------+
# | name | resource | state     |
# +------+----------+-----------+
# | nova | network  | available |
# +------+----------+-----------+
```

### Select an AZ for Compute.

Example:

```bash
openstack availability zone list
# +-----------+-------------+
# | Zone Name | Zone Status |
# +-----------+-------------+
# | cloud-1-a | available   |
# | cloud-1-b | available   |
# | cloud-1-c | available   |
# +-----------+-------------+
```

### Select an AZ for Storage.

Example:

```bash
cinder availability-zone-list
# +------+-----------+
# | Name | Status    |
# +------+-----------+
# | nova | available |
# +------+-----------+
```

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network (a.k.a. Self-Service Network)

In this section a dedicated and private [self-service network][self-service-network] will be setup to host the Kubernetes cluster. OpenStack's self-service network primitive is similar in concept to the [Virtual Private Cloud (VPC) Networks][vpc] created in GCE and AWS.

Create the `kubernetes-the-hard-way` self-service network:

```bash
openstack network create kubernetes-the-hard-way --availability-zone-hint nova
```

### Subnet

A [subnet][self-service-network] must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```bash
openstack subnet create  --network kubernetes-the-hard-way \
  --dns-nameserver 1.1.1.1 --gateway 10.240.0.1 \
  --subnet-range 10.240.0.0/24 kubernetes
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### External Network (a.k.a. Floating IP Network)

Identify the name of your project's external or floating IP network.

For example:

```bash
openstack network list --external --format json | jq '.[0].Name' --raw-output
# tenant-internal-floatingip-net
```

### Router

An OpenStack router must be created to route traffic to and from the private network.

```bash
openstack router create kubernetes-the-hard-way
openstack router set kubernetes-the-hard-way --external-gateway EXTERNAL_NETWORK
openstack router add subnet kubernetes-the-hard-way kubernetes
```

### Security Groups

Create a Security Group that allows internal communication across all protocols:

```bash
openstack security group create kubernetes-the-hard-way-allow-internal

openstack security group rule create \
  --remote-ip 10.240.0.0/24 \
  --protocol any \
  kubernetes-the-hard-way-allow-internal

openstack security group rule create \
  --remote-ip 10.200.0.0/16 \
  --protocol any \
  kubernetes-the-hard-way-allow-internal
```

Create a security group that allows external SSH, ICMP, and HTTPS:

```bash
openstack security group create kubernetes-the-hard-way-allow-external

openstack security group rule create \
  --remote-ip 0.0.0.0/0 \
  --protocol tcp \
  --dst-port 22 \
  kubernetes-the-hard-way-allow-external

openstack security group rule create \
  --remote-ip 0.0.0.0/0 \
  --protocol tcp \
  --dst-port 6443 \
  kubernetes-the-hard-way-allow-external

openstack security group rule create \
  --remote-ip 0.0.0.0/0 \
  --protocol icmp \
  kubernetes-the-hard-way-allow-external
```

> An [external load balancer][lb] will be used to expose the Kubernetes API Servers to remote clients.

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers. Make note of the floating IP for use later when you create the Kubernetes API Server load balancer.

```bash
openstack floating ip create FLOATING_IP_NETWORK_ID --format json | jq '. | {"id": .id, "ip": .floating_ip_address}'
```

Output:

```json
{
  "id": "42cbe563-4371-4dd7-a8ac-78498ee70691",
  "ip": "X.X.X.X"
}
```

Verify the floating IP address was created:

```bash
openstack floating ip show FLOATING_IP_ID --format json
```

Output:

```json
{
  "router_id": null,
  "status": "DOWN",
  "description": "",
  "tags": [],
  "subnet_id": null,
  "dns_name": "",
  "created_at": "2019-01-03T21:44:20Z",
  "updated_at": "2019-01-03T21:44:20Z",
  "dns_domain": "",
  "floating_network_id": "96af047d-5522-4a02-8613-20067c347332",
  "port_details": null,
  "fixed_ip_address": null,
  "floating_ip_address": "X.X.X.X",
  "location": null,
  "revision_number": 1,
  "qos_policy_id": null,
  "project_id": "xyz",
  "port_id": null,
  "id": "42cbe563-4371-4dd7-a8ac-78498ee70691",
  "name": "X.X.X.X"
}
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server 18.04 LTS amd64][ubuntu], which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

> In OpenStack clouds where only Ubuntu Server 16.04 is availble, you can use `apt` to upgrade to 18.04. In OpenStack clouds where neither image is available you will need to add such an image. Upgrading Ubuntu and adding images to OpenStack are outside the scope of this tutorial.

You can identify instance creation prerequisites using these commands:

- `openstack flavor list`
- `openstack image list`
- `openstack keypair list`
- `openstack security group list`

Upstream documentation on how to identify instance prerequisites and create OpenStack instances on the command line is [here][openstack-create-instances].

### Bastion Server

Create one instances that will act as a bastion host. You will SSH through this host to reach those in the private network.

```bash
## customize these
COMPUTE_AZ=cloud-1-c
FLAVOR=1vCPUx2GB
KEYPAIR=my-keypair
STORAGE_AZ=nova
UBUNTU_IMAGE=UBUNTU-18.04

## Instead of creating just bastion-0, you could create
## multiple bastion hosts and place a cloud load balancer
## in front of them, but that's out of scope for this tutorial.
for i in 0; do
  openstack volume create --image ${UBUNTU_IMAGE} \
    --availability-zone ${STORAGE_AZ} --size 20 \
    --bootable bastion-${i}

  openstack server create \
    --availability-zone ${COMPUTE_AZ} --key-name ${KEYPAIR} --volume bastion-${i} \
    --flavor ${FLAVOR} --security-group kubernetes-the-hard-way-allow-external \
    --property cluster=kubernetes-the-hard-way --property pool=bastion \
    --nic net-id=$(openstack network show kubernetes-the-hard-way --format json | jq -r '.id'),v4-fixed-ip=10.240.0.5 \
    bastion-${i}
done
```

Allocate a static IP address that will be attached to the bastion host:

```bash
openstack floating ip create FLOATING_IP_NETWORK_ID --format json | jq '. | {"id": .id, "ip": .floating_ip_address}'
```

Output:

```json
{
  "id": "a9c0c748-5ff9-450f-8a0e-6f5317122a82",
  "ip": "X.X.X.X"
}
```

Verify the floating IP address was created:

```bash
openstack floating ip show FLOATING_IP_ID --format json
```

Output:

```json
{
  "router_id": null,
  "status": "DOWN",
  "description": "",
  "tags": [],
  "subnet_id": null,
  "dns_name": "",
  "created_at": "2019-01-03T21:44:20Z",
  "updated_at": "2019-01-03T21:44:20Z",
  "dns_domain": "",
  "floating_network_id": "96af047d-5522-4a02-8613-20067c347332",
  "port_details": null,
  "fixed_ip_address": null,
  "floating_ip_address": "X.X.X.X",
  "location": null,
  "revision_number": 1,
  "qos_policy_id": null,
  "project_id": "xyz",
  "port_id": null,
  "id": "a9c0c748-5ff9-450f-8a0e-6f5317122a82",
  "name": "X.X.X.X"
}
```

Associate the bastion floating IP with the bastion instance.

```bash
openstack server add floating ip bastion-0 FLOATING_IP
```

Output:

```json
{
  "id": "a9c0c748-5ff9-450f-8a0e-6f5317122a82",
  "ip": "X.X.X.X"
}
```

### Kubernetes Controllers

Create three compute instances that will host the Kubernetes control plane. The flavor you choose for controller instances should at bare minimum include 1vCPU and 1GB RAM.

```bash
## customize these
COMPUTE_AZ=cloud-1-c
FLAVOR=1vCPUx2GB
KEYPAIR=my-keypair
STORAGE_AZ=nova
UBUNTU_IMAGE=UBUNTU-18.04

for i in 0 1 2; do
  openstack volume create --image ${UBUNTU_IMAGE} \
    --availability-zone ${STORAGE_AZ} --size 20 \
    --bootable controller-${i}

  openstack server create \
    --availability-zone ${COMPUTE_AZ} --key-name ${KEYPAIR} --volume controller-${i} \
    --flavor ${FLAVOR} --security-group kubernetes-the-hard-way-allow-internal \
    --property cluster=kubernetes-the-hard-way --property pool=controller \
    --nic net-id=$(openstack network show kubernetes-the-hard-way --format json | jq -r '.id'),v4-fixed-ip=10.240.0.1${i} \
    controller-${i}
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```bash
## customize these
COMPUTE_AZ=cloud-1-c
FLAVOR=1vCPUx2GB
KEYPAIR=my-keypair
STORAGE_AZ=nova
UBUNTU_IMAGE=UBUNTU-18.04

for i in 0 1 2; do
  openstack volume create --image ${UBUNTU_IMAGE} \
    --availability-zone ${STORAGE_AZ} --size 20 \
    --bootable worker-${i}

  openstack server create \
    --availability-zone ${COMPUTE_AZ} --key-name ${KEYPAIR} --volume worker-${i} \
    --flavor ${FLAVOR} --security-group kubernetes-the-hard-way-allow-internal \
    --property cluster=kubernetes-the-hard-way --property pool=worker \
    --property pod-cidr=10.200.${i}.0/24 \
    --nic net-id=$(openstack network show kubernetes-the-hard-way --format json | jq -r '.id'),v4-fixed-ip=10.240.0.2${i} \
    worker-${i}
done
```

### Verification

List the compute instances in your default compute zone:

```bash
openstack server list
# +--------------------------------------+--------------+--------+---------------------------------------------+-------+-----------+
# | ID                                   | Name         | Status | Networks                                    | Image | Flavor    |
# +--------------------------------------+--------------+--------+---------------------------------------------+-------+-----------+
# | 55d63612-331a-47dd-9636-06edb7a3e6d5 | bastion-0    | ACTIVE | kubernetes-the-hard-way=10.240.0.5, X.X.X.X |       | 1vCPUx2GB |
# | c87cb7ad-dc34-4566-b266-92bb9587d918 | worker-2     | ACTIVE | kubernetes-the-hard-way=10.240.0.22         |       | 1vCPUx2GB |
# | b9e65a1f-6244-4bb9-8059-410ad925aa3c | worker-1     | ACTIVE | kubernetes-the-hard-way=10.240.0.21         |       | 1vCPUx2GB |
# | 7d00f75e-2312-483d-b34f-cc302810b23b | worker-0     | ACTIVE | kubernetes-the-hard-way=10.240.0.20         |       | 1vCPUx2GB |
# | df4f6387-2a61-4baa-8f87-67100f28168e | controller-2 | ACTIVE | kubernetes-the-hard-way=10.240.0.12         |       | 1vCPUx2GB |
# | 81774eef-df42-4d78-99c7-c96dc6ad8385 | controller-1 | ACTIVE | kubernetes-the-hard-way=10.240.0.11         |       | 1vCPUx2GB |
# | 40f440cc-c612-4a79-8ad6-1c0e3de71326 | controller-0 | ACTIVE | kubernetes-the-hard-way=10.240.0.10         |       | 1vCPUx2GB |
# +--------------------------------------+--------------+--------+---------------------------------------------+-------+-----------+
```

## Configuring SSH Access

NOTE: Creation and handling of SSH keys is outside the scope of this tutorial.

SSH will be used to configure the controller and worker instances.

Test SSH access to the `bastion-0` instance:

```
ssh -A ubuntu@BASTION_FLOATING_IP
# Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-1006-generic x86_64)
# ...
# Last login: Sun May 13 14:34:27 2018 from X.X.X.X
```

Type `exit` at the prompt to exit the `bastion-0` instance:

```
$USER@bastion-0:~$ exit
# logout
# Connection to X.X.X.X closed
```

You can optionally add a snippet like the following to your `~/.ssh/config` file. Doing so should allow you to automatically proxy through the bastion host to your controller and worker instances.

```
Host bastion-0
  User ubuntu
  Hostname X.X.X.X
  IdentityFile /vagrant/my-keypair.pem

Host controller-0
  User ubuntu
  Hostname 10.240.0.10
  IdentityFile /vagrant/my-keypair.pem
  ProxyCommand ssh bastion-0 -W %h:%p

Host controller-1
  User ubuntu
  Hostname 10.240.0.11
  IdentityFile /vagrant/my-keypair.pem
  ProxyCommand ssh bastion-0 -W %h:%p

Host controller-2
  User ubuntu
  Hostname 10.240.0.12
  IdentityFile /vagrant/my-keypair.pem
  ProxyCommand ssh bastion-0 -W %h:%p

Host worker-0
  User ubuntu
  Hostname 10.240.0.20
  IdentityFile /vagrant/my-keypair.pem
  ProxyCommand ssh bastion-0 -W %h:%p

Host worker-1
  User ubuntu
  Hostname 10.240.0.21
  IdentityFile /vagrant/my-keypair.pem
  ProxyCommand ssh bastion-0 -W %h:%p

Host worker-2
  User ubuntu
  Hostname 10.240.0.22
  IdentityFile /vagrant/my-keypair.pem
  ProxyCommand ssh bastion-0 -W %h:%p
```

If you implement a customized version of the above config, you should be able to SSH from your development machine to `controller-0` like this:

```
ssh controller-0
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)

<!--- Hidden References -->

[az]: https://docs.openstack.org/newton/networking-guide/config-az.html
[lb]: https://docs.openstack.org/mitaka/networking-guide/config-lbaas.html
[openstack-create-instances]: https://docs.openstack.org/ocata/user-guide/cli-launch-instances.html
[self-service-network]: https://docs.openstack.org/newton/install-guide-ubuntu/launch-instance-networks-selfservice.html
[ubuntu]: http://releases.ubuntu.com/18.04/
[vpc]: https://cloud.google.com/compute/docs/vpc/
