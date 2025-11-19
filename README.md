# Kubernetes Minecraft Home Lab
A complete guide and IaC setup for deploying Minecraft on a Raspberry Pi K3s cluster secured behind a FortiGate firewall.

## Prerequisites
**Hardware**

* 2× Raspberry Pi (Pi 4 recommended)
* 1× FortiGate firewall (any small model: 40F/60F/etc.)
* microSD cards (32GB+), power cables, Ethernet cables


## Step 1 - FortiGate Network Setup Guide

In this step we will configure FortiGate so each Raspberry Pi sits in its own subnet, similar to Cloud Availability Zones (AZs).

- Raspberry Pi 1 → Subnet “AZ1” (e.g., 10.10.1.0/24)
- Raspberry Pi 2 → Subnet “AZ2” (e.g., 10.10.2.0/24)
- Allow full communication between subnets
- Provide Internet access for both
- FortiGate acts as the router & firewall
- Home router only gives WAN


**1.** Login to FortiGate web UI

My homework network is `192.168.1.0/24` , I will access FortiGate by `http://192.168.1.99` , the **default username** is `admim` , no password (make sure you change that).

**2.** Change the default admin password and rename the fortiGate device name (i will name mine `Homelab-FortiGate`).

**3.** Configure Interfaces

connect the fortigate to the wan interface But make sure you configured the wan interface to accept http.

**4.** Go to Network → Interfaces

*if you dont see many interfaces you need to remove the interfaces from the lan internal switch first.*

Configure interface 1 (lan1) with 
* IP/Network Mask : 10.10.1.1/24
* Alias : rpi1
* Role: lan
* Addressing mode : manual `10.10.1.1/255.255.255.0`
* DHCP Server : Enabled `10.10.1.2-10.10.1.254`

Configure interface 2 (lan2) with

* IP/Network Mask : 10.10.2.1/24
* Alias : rpi2
* Role: lan
* Addressing mode : manual `10.10.2.1/255.255.255.0`
* DHCP Server : Enabled `10.10.2.2-10.10.2.254`

**5.** Allow LAN → WAN and Wan → LAN traffic

Go to Policy & Objects → IPv4 Policy → Create New

* Name : From Lan1 (rpi1) to Wan
* Incoming Interface : lan1
* Outgoing Interface : fortigate(wan)
* Source : all
* Destination : all
* Schedule : always
* Service : ALL
* Action : ACCEPT
* NAT : enabled

Do the same for wan to lan1

* Name : From Wan to Lan1 (rpi1)
* Incoming Interface : fortigate(wan)
* Outgoing Interface : lan1
* Source : all
* Destination : all
* Schedule : always
* Service : ALL
* Action : ACCEPT
* **NAT : disabled**

Do the same for lan2


Now we need to Allow traffic from Lan1 to Lan2 and Lan2 to Lan1

Go to Policy & Objects → IPv4 Policy → Create New
* Name : From Lan1 (rpi1) to Lan2 (rpi2)
* Incoming Interface : lan1
* Outgoing Interface : lan2
* Source : all
* Destination : all
* Schedule : always
* Service : ALL
* Action : ACCEPT
* NAT : disabled
Do the same for lan2 to lan1


**7.** Add a static route on your home router

You need your home network to know that traffic to 10.10.1.0/24 and 10.10.2.0/24 should go via the FortiGate WAN IP (192.168.217.x).

* Destination: 10.10.1.0/24
* Gateway: 192.168.217.x (FortiGate WAN IP)
* Interface: whatever your router calls your LAN
* Destination: 10.10.2.0/24
* Gateway: 192.168.1.217.x (My FortiGate WAN IP)


if you are on wsl as myself you will need first find your dg
`ip route`
example :
```
default via 172.22.176.1 dev eth0 proto kernel
10.10.1.0/24 via 172.22.176.1 dev eth0
10.10.2.0/24 via 172.22.176.1 dev eth0
10.42.0.0/24 dev cni0 proto kernel scope link src 10.42.0.1
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.22.176.0/20 dev eth0 proto kernel scope link src 172.22.183.53
```

Then add the routes
```
sudo ip route add 10.10.1.0/24 via 172.22.176.1
sudo ip route add 10.10.2.0/24 via 172.22.176.1
```

## Step 2 - Configure Raspberry Pi Device
Follow the official K3s installation guide to set up a multi-node K3s cluster on your Raspberry Pis. Ensure each Pi is assigned to its respective subnet as configured in FortiGate.

First we need to configure the Raspberry Pi SD Card (Do this for each Pi)

1. Flash Raspberry Pi OS Lite onto each microSD card using Raspberry Pi Imager.
Make sure you configure username and password during the setup process in the imager tool for ssh access.
2. plug the SD card into the Raspberry Pi and power it on (wait for couple of minutes to complete first boot).
3. Remove the SD card from the Raspberry Pi and insert it into your computer again.
4. enter to the boot folder of the SD card open the file `cmdline.txt` - 
Follow the official documentation for
[K3S Requirements](https://docs.k3s.io/installation/requirements)

5. plug the SD card into the Raspberry Pi  , connect it to the FortiGate lan1 for the first Pi and lan2 for the second Pi  , and power it on.

*From fortigate monitor → dhcp monitor we will see that each pi recived ip based on his subnet , make sure you create dhcp reservation for both devices.*

### connectivty test
From your computer try to ping and ssh both raspberry pi ips (10.10.1.x and 10.10.2.x) to make sure everything is working fine.

then ssh into the first raspberry pis `ssh ssh rpi@10.10.1.2` and try to ping the second raspberry pi ip to make sure that both devices can communicate with each other.

## Step 3 - K3d Cluster Setup

First we need to install all requirements on both raspberry pis in order to make k3d work fine.
we will have 1 master node and 1 worker node. each raspberry pi will host one node.

**Run inside the raspberry pi 1 and 2.**

1. Enable ip tables.

```
# Enable bridged traffic to be processed by iptables
sudo modprobe br_netfilter

# Make sure it's loaded on boot
echo "br_netfilter" | sudo tee -a /etc/modules-load.d/k8s.conf

# Set required sysctl params
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl settings immediately
sudo sysctl --system
sudo reboot
```

Reconnect to the raspberry pi 1 and 2

2. Install K3S on Raspberry Pi 1 (Master Node)
```
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -
```
3. Check that k3s is running:
`sudo systemctl status k3s`
4. Get the K3S token from the master node:
```
sudo cat /var/lib/rancher/k3s/server/node-token
```
Token Example:
```
K10111800928a5eb106127ad0186a38c568d91d95a505d7358d373f2f64fe946f45::server:ec8da4807902fc88a18bbde627f97797
```
5. ssh into Raspberry Pi 2 (Worker Node) and install K3S using the token from the master node:
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent --server https://10.10.1.2:6443 --token K10111800928a5eb106127ad0186a38c568d91d95a505d7358d373f2f64fe946f45::server:ec8da4807902fc88a18bbde627f97797" sh -s -
```
6. from the raspberry pi 1 (master node) check that the worker node has joined the cluster:
```
kubectl get nodes
```
7. Lable the new node as worker
```
kubectl label node rpi2 node-role.kubernetes.io/worker=worker
```
*The next instructions are optional , in case you want to debug the cluster from the rpi 2.*

8. Create a directory for kubeconfig in raspberry pi 2 (worker node)
`mkdir -p $HOME/.kube`

9. Copy the kubeconfig file from the master node

```scp rpi@10.10.1.2:/etc/rancher/k3s/k3s.yaml $HOME/.kube/config```

10. Change the context to use the worker node's IP

`sed -i 's/127.0.0.1/10.10.1.2/' $HOME/.kube/config`

11. Set permissions for the kubeconfig file
`sudo chown $(id -u):$(id -g) $HOME/.kube/config`

12. Verify kubectl access
kubectl get nodes

13. to access the k3s cluster from your computer, copy the kubeconfig file from the master node to your computer:
```
scp rpi@10.10.1.2:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```
14. Change the context to use the master node's IP

`sed -i 's/127.0.0.1/10.10.1.2/' ~/.kube/config`

