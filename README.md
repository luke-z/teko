# teko

## Multipass

1. Install multipass (choose hyperv or virtualbox)
2. Set bridge network
```
# get network name
multipass networks

# choose connected network (on Windows it's most likely 'Ethernet')
# Check in Control Panel -> Network and Internet -> Network and Sharing Center -> Change adapter settings
multipass set local.bridged-network="<windwos network interface name>"
```

3. Launch the vms (change CPU and RAM according to your system performance)
```
multipass launch -n k3s-master -c 2 -m 4G -d 20G --bridged
```
```
multipass launch -n k3s-worker-* -c 2 -m 4G -d 20G --bridged
```
 
 4. Define networking in the vms
```
# connect to the vms using 
multipass shell <vm name>

# CAREFUL: choose available ips in your network (not in dhcp range).
# to get the right interface for the vm, enter 
ip a
# selected the interface using the correct ip for your host network
# use the following command to see the mac address of your interface:
ip l

# looks like: 00:15:5d:73:e2:90

# open a text editor
sudo nano /etc/netplan/99-multipass.yaml

# now add the following content to the 99-multipass.yaml:
network:
    ethernets:
        <name of ubuntu interface>:
            dhcp4: false
            match:
                macaddress: <mac from ip l>
            set-name: <name of ubuntu interface>
            addresses: [192.168.x.x/24]  # ip from your network
    version: 2
	
	
# Press ctrl+x, then y to safe the changes
# Apply the config with and close the window

sudo netplan apply
```

5. Install k3s on master node
```
# Execute the following commands in Powershell
# for the --node-ip flag, use the IP you set for the master in 99-multipass.yaml

multipass exec k3s-master -- bash -c "curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--node-ip=<IP> --cluster-init --disable servicelb' sh -"
```

5.1 get k3s token
```
multipass exec k3s-master sudo cat /var/lib/rancher/k3s/server/node-token
```

6. Install k3s on all workers
```
# replace IP_MASTER with the master node ip, the TOKEN with the previous token and replace IP_WORKER with the worker IP

multipass exec k3s-worker-* -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://<IP_MASTER>:6443 K3S_TOKEN=<TOKEN> INSTALL_K3S_EXEC='--node-ip=<IP_WORKER>' sh -"
```

7. Install kube-vip (best to use sudo -i), only on master node!
```
# Determine your network interface name for example with ip a
export INTERFACE=<ubuntu network interface>

# Define a virtual IP address (another free IP in your network)
export VIP=<your VIP>

# Install RBAC resources
curl https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
kubectl apply -f /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml

# Run the kube-vip container interactive to generate the daemon manifests and save it in the k3s manfiest folder to auto deploy it
kubectl run -it kube-vip-init  --image=ghcr.io/kube-vip/kube-vip:main --restart=Never --rm -- manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --arp \
    --leaderElection > /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml

# Open the newly generated file and delete the last line in the generated yaml file (pod deleted ...)
# currently there is a bug where v0.7.0 is used, but it does not exist. So make sure to replace every occurence of v.0.7.0 with v.0.6.4 
nano /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml
kubectl apply -f /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml
```

8. Get the kubeconfig and add it to your local config on the host machine
```
# Install chocolatey for Powershell or cmd (on host machine)
https://docs.chocolatey.org/en-us/choco/setup

# Install kubectx (on host machine)
choco install kubens kubectx

# get the kube config (on master node)
cat /etc/rancher/k3s/k3s.yaml

# make a backup of the current kube config (Windows: C:\Users\<username>\.kube\config) and create another config with the content of the previous k3s.yaml. Change the ip to the master node ip 192.168.x.x
```

9. Create the following yaml files on your host machine
``` 
# the address range can be for example 192.168.1.60 - 192.168.1.61.
# Both IPs need to be available and unused in your network
# ipaddresspool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.x.x-192.168.x.x

# l2advertisement.yaml -> replace INTERFACE with the vm interface
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
  interfaces:
  - <INTERFACE>
```

10. Install metallb
```
# Install helm on your host pc
choco install kubernetes-helm

# Add Metallb Repo for helm
helm repo add metallb https://metallb.github.io/metallb
helm repo update
# Install metallb for Services of type LoadBalancer
helm upgrade --install metallb metallb/metallb --namespace metallb-system --create-namespace
# Create ip addresspool
kubectl apply -f ipaddresspool.yml
# Create l2 NIC assignment
kubectl apply -f l2advertisement.yml
```
