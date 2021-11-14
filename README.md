# Setting up Kubernetes on Azure with kubeadm
In this post we share how to setup a High Availability Kubernetes cluster with `kubeadm` using regular VMs running on Microsoft Azure. Managing your own Kubernetes cluster (as opposed to using a managed-Kubernetes service), gives you the most flexibility in configuring Kubernetes. This post comes from a workshop we had with a customer. It can be useful for everyone.

> NB: Azure provides a managed Kubernetes service called "**Azure Kubernetes Service**", aka **AKS**. Please, note we're not using such service for this post.

> NB2: Microsoft provides `aks-engine`, an open-source tool for creating and managing Kubernetes clusters on Azure. Please, note we're not using it.

The steps that are described here are:

- Setup Azure infrastructure
- Prepare VMs
- Setup Kubernetes
- Configure Kubernetes network
- Run Smoke Test
- Cleanup the infrastructure

Most of the steps here are taken from the Kubernetes documentation pages:

- [Bootstrapping clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) 

## Setup Azure infrastructure
As prerequisite, make sure you have the Azuure CLI installed on your workshation. Please, refer to the related [Microsoft Docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

We need a bit of Azure infrastructure to build a highly available Kubernetes cluster on Azure. What we’ll create:

- Virtual Network and Subnet
- Network Security Group
- 3 Control Plane VMs
- 2 Worker VMs
- A load balancer for the Control Plane

> Make sure your Azure subscription provides enough resources quota.

### Create a Global Resource Group
```bash
export REGION_NAME=westeurope
export RESOURCE_GROUP=kubeadm
az group create \
    --name $RESOURCE_GROUP \
    --location $REGION_NAME
```

### Setup networking
```bash
export VNET_NAME=kubeadm
export VNET_ADDRESS=192.168.0.0/16
export SUBNET_NAME=kube
export SUBNET_ADDRESS=192.168.1.0/24
export NSG=kubeadm

az network vnet create \
    --resource-group $RESOURCE_GROUP \
    --name $VNET_NAME \
    --address-prefix $VNET_ADDRESS \
    --subnet-name $SUBNET_NAME \
    --subnet-prefix $SUBNET_ADDRESS
```

### Setup security group
```bash
export NSG=kubeadm

az network nsg create \
    --resource-group $RESOURCE_GROUP \
    --name $NSG

az network nsg rule create \
    --resource-group $RESOURCE_GROUP \
    --nsg-name $NSG \
    --name kubeadmssh \
    --protocol tcp \
    --priority 1000 \
    --destination-port-range 22 \
    --access allow

az network nsg rule create \
    --resource-group $RESOURCE_GROUP \
    --nsg-name $NSG \
    --name kubeadmweb \
    --protocol tcp \
    --priority 1001 \
    --destination-port-range 6443 \
    --access allow

az network vnet subnet update \
    --resource-group $RESOURCE_GROUP \
    --name $SUBNET_NAME \
    --vnet-name $VNET_NAME \
    --network-security-group $NSG

```

### Setup Virtual Machines
First setup VMs for the Control Plane of Kubernetes

```bash
export VM_SIZE=Standard_D2ds_v4
export VM_IMAGE=UbuntuLTS
for i in $(seq -w 0 2); do
    VM_NAME=kube-master-$i;
    az vm create -n $VM_NAME \
        --resource-group $RESOURCE_GROUP \
        --image $VM_IMAGE \
        --vnet-name $VNET_NAME \
        --subnet $SUBNET_NAME \
        --admin-username $USER \
        --ssh-key-value @~/.ssh/id_rsa.pub \
        --size $VM_SIZE \
        --nsg $NSG \
        --public-ip-sku Standard \
        --no-wait;
done
```

Enable IP Forwarding:

```bash
for i in $(seq -w 0 2); do
    VM_NAME=kube-master-$i;
    NIC=${VM_NAME}VMNic;
    az network nic update \
        --resource-group $RESOURCE_GROUP \
        --name $NIC \
        --ip-forwarding=true \
        --no-wait;
done
```

And then create the VMs for the workers

```bash
for i in $(seq -w 0 2); do
    VM_NAME=kube-worker-$i;
    az vm create -n $VM_NAME \
        --resource-group $RESOURCE_GROUP \
        --image $VM_IMAGE \
        --vnet-name $VNET_NAME \
        --subnet $SUBNET_NAME \
        --admin-username $USER \
        --ssh-key-value @~/.ssh/id_rsa.pub \
        --size $VM_SIZE \
        --nsg $NSG \
        --public-ip-sku Standard \
        --no-wait;
done
```

Enable IP Forwarding:

```bash
for i in $(seq -w 0 2); do
    VM_NAME=kube-worker-$i;
    NIC=${VM_NAME}VMNic;
    az network nic update \
        --resource-group $RESOURCE_GROUP \
        --name $NIC \
        --ip-forwarding=true \
        --no-wait;
done
```

### Setup Load Balancer
Setup the Load Balancer for the Control Plane so that it can be accessed as `$DNS_NAME.$REGION_NAME.cloudapp.azure.com`:

```bash
export DNS_NAME=azurekubeadm

az network public-ip create \
    --resource-group $RESOURCE_GROUP \
    --name controlplaneip \
    --sku Standard \
    --dns-name $DNS_NAME 

az network lb create \
    --resource-group $RESOURCE_GROUP \
    --name kubemaster \
    --sku Standard \
    --public-ip-address controlplaneip \
    --frontend-ip-name controlplaneip \
    --backend-pool-name masternodes \
    --no-wait

az network lb probe create \
    --resource-group $RESOURCE_GROUP \
    --lb-name kubemaster \
    --name kubemasterweb \
    --protocol tcp \
    --port 6443

az network lb rule create \
    --resource-group $RESOURCE_GROUP \
    --lb-name kubemaster \
    --name kubemaster \
    --protocol tcp \
    --frontend-port 6443 \
    --backend-port 6443 \
    --frontend-ip-name controlplaneip \
    --backend-pool-name masternodes \
    --probe-name kubemasterweb \
    --disable-outbound-snat true \
    --idle-timeout 15 \
    --enable-tcp-reset true 

```

Add the Control Plane VMs as Load Balancer backends

```bash
for i in $(seq -w 0 2); do
    VM_NAME=kube-master-$i;
    NIC=${VM_NAME}VMNic;
    az network nic ip-config address-pool add \
        --address-pool masternodes \
        --ip-config-name ipconfig${VM_NAME} \
        --nic-name $NIC \
        --resource-group $RESOURCE_GROUP \
        --lb-name kubemaster
done
```

Check all the created resources:

```bash
az resource list --resource-group $RESOURCE_GROUP

Name                    ResourceGroup    Location    Type                                   
----------------------  ---------------  ----------  ---------------------------------------
kubeadm                 kubeadm          westeurope  Microsoft.Network/virtualNetworks
kubeadm                 kubeadm          westeurope  Microsoft.Network/networkSecurityGroups
kube-master-0PublicIP   kubeadm          westeurope  Microsoft.Network/publicIPAddresses
kube-master-1PublicIP   kubeadm          westeurope  Microsoft.Network/publicIPAddresses
kube-master-0VMNic      kubeadm          westeurope  Microsoft.Network/networkInterfaces
kube-master-0           kubeadm          westeurope  Microsoft.Compute/virtualMachines
kube-master-2PublicIP   kubeadm          westeurope  Microsoft.Network/publicIPAddresses
kube-master-1VMNic      kubeadm          westeurope  Microsoft.Network/networkInterfaces
kube-master-2VMNic      kubeadm          westeurope  Microsoft.Network/networkInterfaces
kube-master-1           kubeadm          westeurope  Microsoft.Compute/virtualMachines
kube-master-2           kubeadm          westeurope  Microsoft.Compute/virtualMachines
kube-master-0_OsDisk_1  kubeadm          westeurope  Microsoft.Compute/disks
kube-master-2_OsDisk_1  kubeadm          westeurope  Microsoft.Compute/disks
kube-master-1_OsDisk_1  kubeadm          westeurope  Microsoft.Compute/disks
kube-worker-0PublicIP   kubeadm          westeurope  Microsoft.Network/publicIPAddresses
kube-worker-0VMNic      kubeadm          westeurope  Microsoft.Network/networkInterfaces
kube-worker-1PublicIP   kubeadm          westeurope  Microsoft.Network/publicIPAddresses
kube-worker-0           kubeadm          westeurope  Microsoft.Compute/virtualMachines
kube-worker-2PublicIP   kubeadm          westeurope  Microsoft.Network/publicIPAddresses
kube-worker-1VMNic      kubeadm          westeurope  Microsoft.Network/networkInterfaces
kube-worker-2VMNic      kubeadm          westeurope  Microsoft.Network/networkInterfaces
kube-worker-1           kubeadm          westeurope  Microsoft.Compute/virtualMachines
kube-worker-2           kubeadm          westeurope  Microsoft.Compute/virtualMachines
kube-worker-0_OsDisk_1  kubeadm          westeurope  Microsoft.Compute/disks
kube-worker-2_OsDisk_1  kubeadm          westeurope  Microsoft.Compute/disks
kube-worker-1_OsDisk_1  kubeadm          westeurope  Microsoft.Network/publicIPAddresses
kubemaster              kubeadm          westeurope  Microsoft.Network/loadBalancers

```

## Prepare all the VMs
Once all the resources of Azure infrastructure are in place, prepare all the VMs to run Kubernetes

### Get public IP addresses of the Control Plane VMs

```bash
for i in $(seq -w 0 2); do
    VM_NAME=kube-master-$i;
    export MASTER$i=`az vm list-ip-addresses \
    --resource-group $RESOURCE_GROUP \
    --name $VM_NAME \
    --query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" \
    --output tsv`
done
```

### Get the public IP addresses of the Worker VMs

```bash
for i in $(seq -w 0 2); do
    VM_NAME=kube-worker-$i;
    export WORKER$i=`az vm list-ip-addresses \
    --resource-group $RESOURCE_GROUP \
    --name $VM_NAME \
    --query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" \
    --output tsv`
done
```

### Install Container Runtime
Install `containerd` as container runtime. 

```bash
cat << EOF | tee containerd.conf
overlay
br_netfilter
EOF

cat << EOF | tee 99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

export HOSTS=(${MASTER0} ${MASTER1} ${MASTER2} ${WORKER0} ${WORKER1} ${WORKER2})

for i in "${!HOSTS[@]}"; do
  HOST=${HOSTS[$i]}
  ssh ${USER}@${HOST} -t 'sudo apt update && sudo apt install -y containerd'
  ssh ${USER}@${HOST} -t 'sudo systemctl start containerd && sudo systemctl enable containerd'
  scp containerd.conf ${USER}@${HOST}:
  ssh ${USER}@${HOST} -t 'sudo chown -R root:root containerd.conf && sudo mv containerd.conf /etc/modules-load.d/containerd.conf'
  ssh ${USER}@${HOST} -t 'sudo modprobe overlay && sudo modprobe br_netfilter'
  scp 99-kubernetes-cri.conf ${USER}@${HOST}:
  ssh ${USER}@${HOST} -t 'sudo chown -R root:root 99-kubernetes-cri.conf && sudo mv 99-kubernetes-cri.conf /etc/sysctl.d/99-kubernetes-cri.conf'
  ssh ${USER}@${HOST} -t 'sudo sysctl --system'
done

rm containerd.conf 99-kubernetes-cri.conf

```

Install `crictl`, the command line for working with `containerd`.

```bash
VERSION=v1.22.0
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/${VERSION}/crictl-${VERSION}-linux-amd64.tar.gz
gunzip crictl-${VERSION}-linux-amd64.tar.gz
tar xvf crictl-${VERSION}-linux-amd64.tar

cat << EOF | tee crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
pull-image-on-create: true
EOF

for i in "${!HOSTS[@]}"; do
  HOST=${HOSTS[$i]}
  scp crictl ${USER}@${HOST}:
  ssh ${USER}@${HOST} -t 'sudo chown -R root:root crictl && sudo mv crictl /usr/bin/crictl'
  scp crictl.yaml ${USER}@${HOST}:
  ssh ${USER}@${HOST} -t 'sudo chown -R root:root crictl.yaml && sudo mv crictl.yaml /etc/crictl.yaml'
done

rm crictl*
```

### Install Kubernetes tools
Install `kubectl`, `kubelet`, and `kubeadm` in a recent stable version.

```bash
for i in "${!HOSTS[@]}"; do
  HOST=${HOSTS[$i]}
  ssh ${USER}@${HOST} -t 'sudo apt update'
  ssh ${USER}@${HOST} -t 'sudo apt install -y apt-transport-https ca-certificates curl'
  ssh ${USER}@${HOST} -t 'sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg'
  ssh ${USER}@${HOST} -t 'echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list'
  ssh ${USER}@${HOST} -t 'sudo apt update'
  ssh ${USER}@${HOST} -t 'sudo apt install -y kubelet kubeadm kubectl'
  ssh ${USER}@${HOST} -t 'sudo apt-mark hold kubelet kubeadm kubectl'
done
```

## Setup Kubernetes
Make sure you have the `kubectl` CLI also installed on your workstation.

Set the seed node:

```bash
export SEED=$MASTER0
```

Initialize the seed node:

```bash
export INIT_CMD=$(echo "sudo kubeadm init --control-plane-endpoint=$DNS_NAME.$REGION_NAME.cloudapp.azure.com --upload-certs --v=8")
ssh ${USER}@${SEED} -t ${INIT_CMD}
```

Once the installation completes, export the following envs from the output of the command above:

```bash
export TOKEN=<token>
export CERTIFICATE_KEY=<certificate-key>
export CA_CERTIFICATE_HASH=<discovery-token-ca-cert-hash>
```

Copy the kubeconfig file from the seed node to your workstation:

```bash
  ssh ${USER}@${SEED} -t 'sudo cp -i /etc/kubernetes/admin.conf .'
  ssh ${USER}@${SEED} -t 'sudo chown $(id -u):$(id -g) admin.conf'
  mkdir -p $HOME/.kube
  scp ${USER}@${SEED}:admin.conf $HOME/.kube/$DNS_NAME.$REGION_NAME.cloudapp.azure.com
```

and check the status of the Kubernetes cluster

```bash
export KUBECONFIG=$HOME/.kube/$DNS_NAME.$REGION_NAME.cloudapp.azure.com
kubectl cluster-info

Kubernetes control plane is running at https://azurekubeadm.westeurope.cloudapp.azure.com:6443
CoreDNS is running at https://azurekubeadm.westeurope.cloudapp.azure.com:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

```

Join the remaining control plane nodeS:

```bash
export JOIN_MASTER_CMD=$(echo "sudo kubeadm join $DNS_NAME.$REGION_NAME.cloudapp.azure.com:6443 --control-plane --token=$TOKEN --discovery-token-ca-cert-hash=$CA_CERTIFICATE_HASH --certificate-key=$CERTIFICATE_KEY --v=8")

MASTERS=(${MASTER1} ${MASTER2})
for i in "${!MASTERS[@]}"; do
  MASTER=${MASTERS[$i]}
  ssh ${USER}@${MASTER} -t ${JOIN_MASTER_CMD};
done
```

Join all the worker nodes:

```bash
export JOIN_WORKER_CMD=$(echo "sudo kubeadm join $DNS_NAME.$REGION_NAME.cloudapp.azure.com:6443 --token=$TOKEN --discovery-token-ca-cert-hash=$CA_CERTIFICATE_HASH --v=8")

WORKERS=(${WORKER0} ${WORKER1} ${WORKER2})
for i in "${!WORKERS[@]}"; do
  WORKER=${WORKERS[$i]}
  ssh ${USER}@${WORKER} -t ${JOIN_WORKER_CMD};
done
```

Check the cluster has formed:

```bash
kubectl get nodes

NAME            STATUS     ROLES                  AGE     VERSION
kube-master-0   NotReady   control-plane,master   41m     v1.22.3
kube-master-1   NotReady   control-plane,master   18m     v1.22.3
kube-master-2   NotReady   control-plane,master   17m     v1.22.3
kube-worker-0   NotReady   <none>                 3m29s   v1.22.3
kube-worker-1   NotReady   <none>                 3m22s   v1.22.3
kube-worker-2   NotReady   <none>                 3m15s   v1.22.3
```

Cluster nodes are still in a `NotReady` state because of the missing CNI component.

## Configure Kubernetes Network
Network plugins in Kubernetes come in a few flavors:

- CNI plugins: eg. Flannel, Calico, Cilium
- kubenet

In this post we're going to setup a basic configuration of Calico. Please, refer to [Calico on Azure](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/azure) for more about installation options.

> *TL;DR:* use Calico in VXLAN mode by setting CALICO_IPV4POOL_VXLAN=always.


```bash
kubectl apply -f k8s-cni/calico-crd.yaml
kubectl apply -f k8s-cni/calico.yaml
```

And check all the nodes are now in `Ready` state

```bash
kubectl get nodes
NAME            STATUS     ROLES                  AGE   VERSION
kube-master-0   Ready      control-plane,master   14h   v1.22.3
kube-master-1   Ready      control-plane,master   13h   v1.22.3
kube-master-2   Ready      control-plane,master   13h   v1.22.3
kube-worker-0   Ready      <none>                 13h   v1.22.3
kube-worker-1   Ready      <none>                 13h   v1.22.3
kube-worker-2   Ready      <none>                 13h   v1.22.3
```

## Run Smoke Test
To verify everything works great, let’s run a small little sample webapp on the cluster.

### Deployment
Deploy a `apache` application on the tenant cluster

```bash
kubectl create deployment apache --image=httpd:latest
```

and check the `apache` pod gets scheduled

```bash
kubectl get pods
```

### Port Forwarding
Verify the ability to access applications remotely using port forwarding.

Retrieve the full name of the `apache` pod:

```bash
POD_NAME=$(kubectl get pods -l app=apache -o jsonpath="{.items[0].metadata.name}")
```

Forward port 8080 on your local machine to port 80 of the `apache` pod:

```bash
kubectl port-forward $POD_NAME 8080:80
```

In a new terminal make an HTTP request using the forwarding address:

```bash
curl --head http://127.0.0.1:8080
```

Switch back to the previous terminal and stop the port forwarding to the `apache` pod.

### Logs
Verify the ability to retrieve container logs.

Print the `apache` pod logs:

```bash
kubectl logs $POD_NAME
```

### Tunnel
Verify the ability to execute commands in a container.

Print the `apache` version by executing the `apache -v` command in the `apache` container:

```bash
kubectl exec -ti $POD_NAME -- apachectl -v
```

### Services
Verify the ability to expose applications using a service.

Expose the `apache` deployment using a `NodePort` service:

```bash
kubectl expose deployment apache --port 80 --type NodePort
```

Retrieve the node port assigned to the `apache` service:

```bash
NODE_PORT=$(kubectl get svc apache \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Open the Node Port on the Azure infrastructure

```bash
az network nsg rule create \
    --resource-group $RESOURCE_GROUP \
    --nsg-name $NSG \
    --name sample-node-port \
    --protocol tcp \
    --priority 1002 \
    --destination-port-range $NODE_PORT \
    --access allow
```

Check if the service is reachable from all the nodes:

```bash

for i in "${!WORKERS[@]}"; do
  WORKER=${WORKERS[$i]}
  curl -I http://${WORKER}:${NODE_PORT};
done
```

## Cleanup the infrastructure
```bash
az group delete -n $RESOURCE_GROUP --yes --no-wait
```




