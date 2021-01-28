# Red Hat Ceph 4 micro-cluster on libvirt

## Installation

Define network
```bash
virsh net-define --file ceph-infra/network/ceph.xml && virsh net-start ceph
```

Extend your hosts file
```bash
cat ceph-infra/network/hosts-add >> /etc/hosts
```

Download RHEl 8.3 ISO to ceph-infra/iso

Configure `ceph-infra/anaconda-ks.cfg` with your credentials for subscription-manager.

Deploy RHEL 8.3 nodes
```bash
for i in {1..3}; do virt-install --name=ceph-0$i --cdrom=ceph-infra/iso/rhel-8.3-x86_64-dvd.iso --disk=ceph-infra/disks/ceph-$i-0.qcow2,size=30 --disk=ceph-infra/disks/ceph-$i-1.qcow2,size=10 --disk=ceph-infra/disks/ceph-$i-2.qcow2,size=10 --disk=ceph-infra/disks/ceph-$i-3.qcow2,size=10 --graphics none --vcpus=4 --ram=4096 --location=ceph-infra/iso/rhel-8.3-x86_64-dvd.iso --network network=ceph,mac=12:34:56:00:99:0$i  --os-type=linux --os-variant=rhel8.3 --initrd-inject=ceph-infra/anaconda-ks.cfg --noautoconsole --extra-args="ks=file:/anaconda-ks.cfg ip=dhcp console=ttyS0,115200n8 serial ksdevice=";done
```

Once the VMs are ready, start them
```bash
for i in {1..3}; do virsh start ceph-0$i; done
```

Deploy HAProxy for the dashboard
```bash
podman kill haproxy-ceph; podman rm haproxy-ceph; podman run -d --name haproxy-ceph  -v ./ceph-infra/loadbalancer/haproxy.cfg:/opt/haproxy.cfg:z --net host -p 8443:8443 quay.io/aerdei-redhat.com/fedora-toolkit:v0.0.2 /usr/sbin/haproxy -f /opt/haproxy.cfg -d
```

Install ceph-ansible
```bash
dnf install ceph-ansible -y
```

Copy config files to the installer:
```
cp -r ceph-ansible/* /usr/share/ceph-ansible
```

Create ceph-ansible-keys dir:
```bash
mkdir ~/ceph-ansible-keys
```

Configure `/usr/share/ceph-ansible/group_vars/all.yml` with your Red Hat service account and token.

Copy SSH keys to the Ceph nodes
```bash
for i in {1..3};do sshpass -p r3dh4t1! ssh-copy-id ceph-0$i; done

```
Run installer
```
cd /usr/share/ceph-ansible
ansible-playbook -i inventory site-container.yml
```

## Cleanup
```bash
for i in {1..3};do virsh destroy ceph-0$i; virsh undefine ceph-0$i --storage vda,vdb,vdc,vdd; done
```

## Notes
It probably goes without saying that under no circumstances should you ever consider using this solution in a production-like environment.
