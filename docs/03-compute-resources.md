# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [availability zone](https://cloud.yandex.com/en-ru/docs/overview/concepts/geo-scope).

## Preparing SSH-keys

Be sure that you have pair of ssh-keys in your home directory (`~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`).

If you don't have one, generate a pair of ssh-keys.
```
ssh-keygen
```

During generation be sure that your pair will be stored in default path (`~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`).
```
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/username/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/username/.ssh/id_rsa
Your public key has been saved in /Users/username/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:5M/3XTcFnZkERLvsoHtsgmTUP4emKEJpRKsQ9vuNjI0 username@username.local
The key's randomart image is:
+---[RSA 3072]----+
|            o+.. |
|.. .          o.+|
|..o .   o    ..+.|
|.  +   + .  . .. |
|. o o . S ...o  .|
| . =   o o.=o.  .|
|  o * = o.* +. .o|
|   E * + o.= . .+|
|    . .  .+   . .|
+----[SHA256]-----+
```

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://cloud.yandex.com/en-ru/docs/vpc/concepts/) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```
yc vpc network create --name kubernetes-the-hard-way --description "Main network"
```

A [subnet](https://cloud.yandex.com/en-ru/docs/vpc/concepts/network) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```
yc vpc subnet create --name kubernetes \
  --description "Kubernetes subnet" \
  --network-name kubernetes-the-hard-way \
  --zone ru-central1-a \
  --range 10.240.0.0/24
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Kubernetes Public IP Address

Create a network load balancer with simple listener on the port tcp:6443. There will be attached target groups for this balancer.

```
yc load-balancer network-load-balancer create \
  --name kubernetes-load-balancer \
  --listener name=kubernetes-listener,external-ip-version=ipv4,port=6443,target-port=6443
```

Verify the `kubernetes-load-balancer` was created in your default compute region:

```
yc load-balancer network-load-balancer list
```

> output
```
+----------------------+--------------------------+-------------+----------+----------------+------------------------+----------+
|          ID          |           NAME           |  REGION ID  |   TYPE   | LISTENER COUNT | ATTACHED TARGET GROUPS |  STATUS  |
+----------------------+--------------------------+-------------+----------+----------------+------------------------+----------+
| enpcq3mn52tabordart2 | kubernetes-load-balancer | ru-central1 | EXTERNAL |              1 |                        | INACTIVE |
+----------------------+--------------------------+-------------+----------+----------------+------------------------+----------+
```

Also you can verify that load-balancer automaticly created a Public IP:

```
yc vpc address list
```

> output
```
+----------------------+------+--------------+----------+------+
|          ID          | NAME |   ADDRESS    | RESERVED | USED |
+----------------------+------+--------------+----------+------+
| e9bd2dsn9g3rg7lipe2u |      | 51.250.78.46 | false    | true |
+----------------------+------+--------------+----------+------+
```

Write down given ID of Public IP to update it.

* rename:
`yc vpc address update <ID_of_public_IP> --new-name kubernetes-the-hard-way`
* make it static:
`yc vpc address update <ID_of_public_IP> --reserved=true`

Verify the result:

```
yc vpc address list
```

> output
```
+----------------------+-------------------------+--------------+----------+------+
|          ID          |          NAME           |   ADDRESS    | RESERVED | USED |
+----------------------+-------------------------+--------------+----------+------+
| e9bd2dsn9g3rg7lipe2u | kubernetes-the-hard-way | 51.250.78.46 | true     | true |
+----------------------+-------------------------+--------------+----------+------+
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  yc compute instance create \
      --async \
      --name controller-${i} \
      --hostname controller-${i} \
      --zone $(yc config get compute-default-zone) \
      --cores 2 --memory 8 \
      --create-boot-disk size=30,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
      --network-interface subnet-name=kubernetes,nat-ip-version=ipv4,ipv4-address=10.240.0.1${i} \
      --labels type=controller,project=kubernetes-the-hard-way \
      --ssh-key ~/.ssh/id_rsa.pub
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  yc compute instance create \
      --async \
      --name worker-${i} \
      --hostname worker-${i} \
      --zone $(yc config get compute-default-zone) \
      --cores 2 --memory 8 \
      --create-boot-disk size=30,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
      --network-interface subnet-name=kubernetes,nat-ip-version=ipv4,ipv4-address=10.240.0.2${i} \
      --metadata pod-cidr=10.200.${i}.0/24 \
      --labels type=worker,project=kubernetes-the-hard-way \
      --ssh-key ~/.ssh/id_rsa.pub
done
```

### Verification

List the compute instances in your default compute zone:

```
yc compute instance list
```

> output

```
+----------------------+--------------+---------------+---------+---------------+-------------+
|          ID          |     NAME     |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP |
+----------------------+--------------+---------------+---------+---------------+-------------+
| fhm1u4jenlsv3qnbgvkj | controller-2 | ru-central1-a | RUNNING | 51.250.65.68  | 10.240.0.12 |
| fhm6sollt8c146g2jv8j | worker-2     | ru-central1-a | RUNNING | 51.250.69.241 | 10.240.0.22 |
| fhmbs7jh6k56nce2p49n | worker-1     | ru-central1-a | RUNNING | 51.250.74.55  | 10.240.0.21 |
| fhmdto47vsl0i2o6t2tl | controller-1 | ru-central1-a | RUNNING | 51.250.76.24  | 10.240.0.11 |
| fhmhrklrp2ke4o9cs1ft | worker-0     | ru-central1-a | RUNNING | 51.250.69.211 | 10.240.0.20 |
| fhmt585tist9n6lmi4nr | controller-0 | ru-central1-a | RUNNING | 51.250.68.231 | 10.240.0.10 |
+----------------------+--------------+---------------+---------+---------------+-------------+
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances.

> Ensure you have pair of ssh-keys as described in the [Preparing SSH-keys](03-compute-resources.md#Preparing-SSH-keys).

Save variables with instances' IPs. You have to do it every time when you want to connect to instances (if you closed and opened console).
```
for i in 0 1 2; do
  export CONTROLLER${i}_IP=$(yc compute instance get --name controller-${i} --format json | jq -r '.network_interfaces[0].primary_v4_address.one_to_one_nat.address')
  export WORKER${i}_IP=$(yc compute instance get --name worker-${i} --format json | jq -r '.network_interfaces[0].primary_v4_address.one_to_one_nat.address')
done
```

Test SSH access to the `controller-0` compute instances:

```
ssh yc-user@${CONTROLLER0_IP}
```

If this is your first time connecting to a compute instance SSH you will see the question about trusting this server. Agree by typing **"yes"** to continue:

```
The authenticity of host '51.250.69.211 (51.250.69.211)' can't be established.
ED25519 key fingerprint is SHA256:b4/j5O1usG9qHYOstxJWrkRjRkhFYsXgcxpbYdRKdeM.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

After you have been logged into the `controller-0` instance:

```
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-97-generic x86_64)
...
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
yc-user@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XX.XX.XX closed.
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
