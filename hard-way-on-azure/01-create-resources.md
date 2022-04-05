# Kubernetes the Hard Way on Azure

Original: https://github.com/ivanfioravanti/kubernetes-the-hard-way-on-azure

## Create RG

```
az group create --name kubernetes --location japaneast
```

## Install tools

### cfssl

```
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
chmod +x cfssl cfssljson
mkdir ~/.local/bin
mv cfssl cfssljson ~/.local/bin/
```

## Provisioning Compute Resources

### VNet

```
az network vnet create -g kubernetes \
  -n kubernetes-vnet \
  --address-prefix 10.240.0.0/24 \
  --subnet-name kubernetes-subnet
```

### NSG

```
az network nsg create -g kubernetes -n kubernetes-nsg

az network vnet subnet update -g kubernetes \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --network-security-group kubernetes-nsg

az network nsg rule create -g kubernetes \
  -n kubernetes-allow-ssh \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 22 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1000

az network nsg rule create -g kubernetes \
  -n kubernetes-allow-api-server \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 6443 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1001
```

```
% az network nsg rule list -g kubernetes --nsg-name kubernetes-nsg --query "[].{Name:name, \
  Direction:direction, Priority:priority, Port:destinationPortRange}" -o table
Name                         Direction    Priority    Port
---------------------------  -----------  ----------  ------
kubernetes-allow-ssh         Inbound      1000        22
kubernetes-allow-api-server  Inbound      1001        6443
```

### LB + Kubernetes Public IP Address

```
az network lb create -g kubernetes \
  -n kubernetes-lb \
  --backend-pool-name kubernetes-lb-pool \
  --public-ip-address kubernetes-pip \
  --public-ip-address-allocation static
```

```
% az network public-ip list --query="[?name=='kubernetes-pip'].{ResourceGroup:resourceGroup, \
  Region:location,Allocation:publicIpAllocationMethod,IP:ipAddress}" -o table
ResourceGroup    Region     Allocation    IP
---------------  ---------  ------------  --------------
kubernetes       japaneast  Static        XX.XXX.XXX.XXX
```

### Virtual Machines

Check latest image

```
% az vm image list --location japaneast --publisher Canonical --offer UbuntuServer --sku 18.04-LTS --all -o table
Offer         Publisher    Sku        Urn                                               Version
------------  -----------  ---------  ------------------------------------------------  ---------------
  ...
UbuntuServer  Canonical    18.04-LTS  Canonical:UbuntuServer:18.04-LTS:18.04.202201180  18.04.202201180
```

```
UBUNTULTS="Canonical:UbuntuServer:18.04-LTS:18.04.202201180"

# controller
az vm availability-set create -g kubernetes -n controller-as
for i in 0 1 2; do
    echo "[Controller ${i}] Creating public IP..."
    az network public-ip create -n controller-${i}-pip -g kubernetes > /dev/null

    echo "[Controller ${i}] Creating NIC..."
    az network nic create -g kubernetes \
        -n controller-${i}-nic \
        --private-ip-address 10.240.0.1${i} \
        --public-ip-address controller-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding \
        --lb-name kubernetes-lb \
        --lb-address-pools kubernetes-lb-pool > /dev/null

    echo "[Controller ${i}] Creating VM..."
    az vm create -g kubernetes \
        -n controller-${i} \
        --image ${UBUNTULTS} \
        --nics controller-${i}-nic \
        --availability-set controller-as \
        --nsg '' \
        --admin-username 'kuberoot' \
        --generate-ssh-keys > /dev/null
done

# worker
az vm availability-set create -g kubernetes -n worker-as
for i in 0 1; do
    echo "[Worker ${i}] Creating public IP..."
    az network public-ip create -n worker-${i}-pip -g kubernetes > /dev/null

    echo "[Worker ${i}] Creating NIC..."
    az network nic create -g kubernetes \
        -n worker-${i}-nic \
        --private-ip-address 10.240.0.2${i} \
        --public-ip-address worker-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding > /dev/null

    echo "[Worker ${i}] Creating VM..."
    az vm create -g kubernetes \
        -n worker-${i} \
        --image ${UBUNTULTS} \
        --nics worker-${i}-nic \
        --tags pod-cidr=10.200.${i}.0/24 \
        --availability-set worker-as \
        --nsg '' \
        --generate-ssh-keys \
        --admin-username 'kuberoot' > /dev/null
done
```

```
% az vm list -d -g kubernetes -o table
Name          ResourceGroup    PowerState    PublicIps       Fqdns    Location    Zones
------------  ---------------  ------------  --------------  -------  ----------  -------
controller-0  kubernetes       VM running    XX.XXX.XXX.XXX           japaneast
controller-1  kubernetes       VM running    XX.XXX.XXX.XXX           japaneast
controller-2  kubernetes       VM running    XX.XXX.XXX.XXX           japaneast
worker-0      kubernetes       VM running    XX.XXX.XXX.XXX           japaneast
worker-1      kubernetes       VM running    XX.XXX.XXX.XXX           japaneast
```

Create SSH config

```
az vm list -d -g kubernetes -o json | jq -r '.[] | "Host " + .name + "\n  User kuberoot\n  hostname " + .publicIps + "\n"' >> ~/.ssh/config
```
