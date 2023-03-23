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

1) Extracting openshift-baremetal-install

```
export VERSION=latest-4.8
export RELEASE_IMAGE=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/release.txt | grep 'Pull From: quay.io' | awk -F ' ' '{print $3}')
export cmd=openshift-baremetal-install
export pullsecret_file=~/pullsecret.txt

curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/openshift-client-linux.tar.gz | tar zxvf - oc
sudo cp oc /usr/local/bin
oc adm release extract --registry-config "${pullsecret_file}" --command=$cmd --to . ${RELEASE_IMAGE}
sudo cp ./openshift-baremetal-install /usr/local/bin
```

2) Downloading coreos-installer 

```
wget https://mirror.openshift.com/pub/openshift-v4/clients/coreos-installer/v0.8.0-3/coreos-installer
cp ./coreos-installer /usr/local/bin && chmod +x /usr/local/bin/coreos-installer
```

3) Downloading rhcos live media 

```
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/4.8.2/rhcos-4.8.2-x86_64-live.x86_64.iso
```

4) Preparing the host

```
yum groupinstall "Virtualization Host" -y
yum install virt-install libvirt-client -y
systemctl enable --now libvirtd.service
```

5) Creating the network 

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
  <mac address='52:54:00:fd:be:d0'/>
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

Configuring DNS using dnsmasq and NetworkManager

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

6) Creating install-config.yaml

Getting the secret at the following link: 

https://console.redhat.com/openshift/create/local

click on "Copy pull secret" button in the Pull Secret section. 

```
mkdir /opt/openshift 
cd /opt/openshift/
mkdir /opt/openshift/deploy
```

Create the install-config.yaml file with the following template: 

CLUSTER_NAME: fajlinux
BASE_DOMAIN: local 


vi deploy/install-config.yaml

``` 
apiVersion: v1beta4
baseDomain: $BASE_DOMAIN
metadata:
  name: $CLUSTER_NAME
networking:
  networkType: OVNKubernetes
  machineCIDR: 192.168.130.0/24
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
platform:
  none: {}
BootstrapInPlace:
  InstallationDisk: /dev/vda
pullSecret: "PASTE THE SECRET CONTENT HERE"
sshKey: "PASTE THE SSH PUBLIC KEY HERE"
EOF
```

7) Generating single node media 

```
openshift-baremetal-install --dir=sno create single-node-ignition-config
coreos-installer iso ignition embed -fi /root/sno/bootstrap-in-place-for-live-iso.ign /root/rhcos-4.8.2-x86_64-live.x86_64.iso
cp -rf rhcos-4.8.2-x86_64-live.x86_64.iso /var/lib/libvirt/images/rhcos-4.8.2-x86_64-live.x86_64.iso
```


8) Installing Single Node Cluster 

Based on minimal requirements 
https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-preparing-to-install-sno.html


virsh net-update default add ip-dhcp-host "<host mac='52:54:00:65:aa:da' name='cluster.fajlinux.local' ip='192.168.130.11'/>" --live --config

```
virt-install --name="openshift-sno" \
    --vcpus=4 \
    --ram=12 \
    --disk path=/var/lib/libvirt/images/master-snp.qcow2,bus=sata,size=120 \
    --network network=sno,model=virtio \
    -m 52:54:00:65:aa:da \
    --boot menu=on \
    --graphics vnc --console pty,target_type=serial --noautoconsole \
    --cpu host-passthrough \
    --cdrom /var/lib/libvirt/images/rhcos-4.8.2-x86_64-live.x86_64.iso
```

Let's wait around 40 to 60 minutes. 


## Troubleshotting commands: 

You can check the progress:  ( expect lots of erros and ignore it )

```
openshift-baremetal-install wait-for install-complete --dir /opt/openshift/deploy

export KUBECONFIG=/root/sno/auth/kubeconfig

oc get nodes
oc get co 
oc get clusterversion
```
