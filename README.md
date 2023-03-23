# openshift-sno-libvirt
Openshift running as single node with libvirt kvm


# Libvirt configuration - Openshift Single Node SNO 

I intend to create an Ansible playbook project in the future. 
just created a recipe as of now. 


## Environment

My environment is below:

```
RHEL8.5
libvirtd (libvirt) 8.0.0 
libvirt network: default - Range 192.168.130.0/24 
Domain and Single Node IP: *.fajlinux.local 192.168.130.11 
```

## Host setup

1) Libvirt installing

```
yum groupinstall "Virtualization Host" -y
yum install virt-install libvirt-client -y
systemctl enable --now libvirtd.service
```

2) Libvirt networking setup 

vi net.xml 

```
<network connections='1'>
  <name>sno</name>
  <uuid>49eee855-d342-46c3-9ed3-b8d1758814cd</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='sno' stp='on' delay='0'/>
  <ip family='ipv4' address='192.168.130.1' prefix='24'>
  </ip>
</network>
```

Setting up the network 

```
virsh net-define net.xml

[root@compute1 ~]# virsh net-list 
 Name      State    Autostart   Persistent
--------------------------------------------
 sno       active   yes         yes
 default   active   yes         yes
 labnet1   active   yes         yes
 
```

3) Configuring DNS using dnsmasq and NetworkManager

Domain: fajlinux.local 
SNO NODE IP: 192.168.130.11
Name Servers: 192.168.15.1, 8.8.8.8 

```
yum install dnsmasq

echo -e "[main]\ndns=dnsmasq" | sudo tee /etc/NetworkManager/conf.d/openshift.conf
echo listen-address=127.0.0.1 > /etc/NetworkManager/dnsmasq.d/openshift.conf
echo bind-interfaces >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo server=192.168.15.1 >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo server=8.8.8.8 >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo address=/fajlinux.local/192.168.130.11 >> /etc/NetworkManager/dnsmasq.d/openshift.conf


systemctl reload NetworkManager

nslookup master.fajlinux.local

```




## Openshift Setup


1) Extracting openshift-baremetal-install

I'm working in the directory path /opt/openshift

```

mkdir /opt/openshift
cd /opt/openshift
```

Download the necessary files and binaries for Openshift 4.10

```

export VERSION=latest-4.10
export RELEASE_IMAGE=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/release.txt | grep 'Pull From: quay.io' | awk -F ' ' '{print $3}')

curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/openshift-client-linux.tar.gz
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/openshift-install-linux.tar.gz 
tar -xvf openshift-install.tar.gz 
tar -xvf openshift-client-linux.tar.gz

```

2) Downloading coreos-installer 

```

wget https://mirror.openshift.com/pub/openshift-v4/clients/coreos-installer/v0.8.0-3/coreos-installer
cp ./coreos-installer /usr/local/bin && chmod +x /usr/local/bin/coreos-installer

```

3) Downloading rhcos live media 


```

 wget -v https://rhcos.mirror.openshift.com/art/storage/releases/rhcos-4.10/410.84.202210040010-0/x86_64/rhcos-410.84.202210040010-0-live.x86_64.iso

```


4) Creating install-config.yaml
5) 

Getting the secret at the following link: 

https://console.redhat.com/openshift/create/local

click on "Copy pull secret" button in the Pull Secret section. 

```

mkdir /opt/openshift/deploy

```

Create the install-config.yaml file with the following template: 

CLUSTER_NAME: fajlinux
BASE_DOMAIN: local 


vi deploy/install-config.yaml

```

apiVersion: v1
baseDomain: local
metadata:
  name: fajlinux
networking:
  networkType: OVNKubernetes
  machineCIDR: 192.168.130.0/24
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23 
  serviceNetwork:
  - 172.30.0.0/16
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
platform:
  none: {}
BootstrapInPlace:
  InstallationDisk: /dev/sda
pullSecret: "PASTE THE SECRET CONTENT HERE"
sshKey: "PASTE THE SSH PUBLIC KEY HERE"

```

5) Generating single node media 

```
openshift-baremetal-install --dir=deploy create single-node-ignition-config
coreos-installer iso ignition embed -fi /opt/openshift/deploy/bootstrap-in-place-for-live-iso.ign /opt/openshift/rhcos-4.8.2-x86_64-live.x86_64.iso
cp -rf rhcos-4.8.2-x86_64-live.x86_64.iso /var/lib/libvirt/images/rhcos-sno-4.8.2-x86_64-live.x86_64.iso
```

## Installing Single Node Cluster 

Based on minimal requirements 
https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-preparing-to-install-sno.html


```
# Create the ip reservation first
coreos-installer iso kargs modify -a "ip=192.168.130.11::192.168.130.1:255.255.255.0:fajlinux:ens3:off:192.168.130.1" /var/lib/libvirt/images/rhcos-sno-4.8.2-x86_64-live.x86_64.iso

# Demonstration of the command 
coreos-installer iso kargs modify -a "ip=${ip}::${gateway}:${netmask}:${hostname}:${interface}:none:${nameserver}" FILENAME.


# Create the vm
virt-install --name="openshift-sno" \
    --vcpus=4 \
    --ram=12288 \
    --disk path=/var/lib/libvirt/images/master-snp.qcow2,bus=sata,size=120 \
    --network network=sno,model=virtio \
    --boot menu=on \
    --graphics vnc --console pty,target_type=serial --noautoconsole \
    --cpu host-passthrough \
    --cdrom /var/lib/libvirt/images/rhcos-sno-4.8.2-x86_64-live.x86_64.iso
```

Let's wait around 40 to 60 minutes. 


## Troubleshotting commands: 

You can check the progress:  ( expect lots of erros and ignore it )

```
openshift-baremetal-install wait-for install-complete --dir /opt/openshift/deploy

export KUBECONFIG=/opt/openshift/deploy/auth/kubeconfig

oc get nodes
oc get co 
oc get clusterversion
```
