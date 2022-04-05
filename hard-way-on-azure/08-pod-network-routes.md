# Provisioning Pod Network Routes

Original: https://github.com/ivanfioravanti/kubernetes-the-hard-way-on-azure/blob/master/docs/11-pod-network-routes.md

## The Routing Table

```
for instance in worker-0 worker-1; do
  PRIVATE_IP_ADDRESS=$(az vm show -d -g kubernetes -n ${instance} --query "privateIps" -otsv)
  POD_CIDR=$(az vm show -g kubernetes --name ${instance} --query "tags" -o tsv)
  echo $PRIVATE_IP_ADDRESS $POD_CIDR
done
```

## Routes

```
az network route-table create -g kubernetes -n kubernetes-routes

az network vet subnet update -g kubernetes \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --route-table kubernetes-routesn

for i in 0 1; do
az network route-table route create -g kubernetes \
  -n kubernetes-route-10-200-${i}-0-24 \
  --route-table-name kubernetes-routes \
  --address-prefix 10.200.${i}.0/24 \
  --next-hop-ip-address 10.240.0.2${i} \
  --next-hop-type VirtualAppliance
done

az network route-table route list -g kubernetes --route-table-name kubernetes-routes -o table
```
