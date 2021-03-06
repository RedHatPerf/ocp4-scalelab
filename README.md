
# OCP 4.x JetSki IPI install steps
Check the [jetski.md](JETSKI.md) markdown for instructions

# OCP 4.x bare-metal install guide in SCALE lab

The setup requires coporate DHCP on public interfaces (10.x.x.x) not seeding the domain nameservers. This should be eventually avoided when [RFE-240](https://jira.coreos.com/browse/RFE-240) gets incorporated.

TODO: From my experience the major problem following this tutorial is replacing all IPs and names correctly. The customization should be templated.

## Local machine setup

The steps below basically follow the [bare-metal install guide](https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html#installation-operators-config_installing-bare-metal):

Generate a new RSA key pair and store them in you `~/.ssh/`.

```
ssh-keygen -t rsa -b 4096 -N '' -f core_rsa
```

Download installer and pull secret from [UPI downloads page](https://cloud.redhat.com/openshift/install/metal/user-provisioned) and extract them.

Place the pull secret and your public key into `install-config.yaml`; change replica counts and `metadata.name` as needed.

Create a new directory and copy the updated `install-config.yaml` in there. Then run `openshift-installer`:

```
mkdir cloud
cp install-config.yaml cloud/
./openshift-install create ignition-configs --dir cloud --log-level debug
```

The created directory should contain `bootstrap.ign`, `master.ign` and `worker.ign`.

Point your machine to the bastion node public IP: if you're using `dns=dnsmasq` in `/etc/NetworkManager/NetworkManager.conf` like me just add `scalelab.conf` to `/etc/NetworkManager/dnsmasq.d/` and fix the IPs. Restart NetworkManager aftewards.

## Setup Bastion node

Ssh into the bastion onde machine using `root` / `100yard-`, add your pubkey to authorized_keys for convenience and install necessary software:

```
yum install -y bind dhcp haproxy tftp-server
```

Ssh into the other nodes used for Openshift nodes and record MAC addreses. I have used the `p2p3` interface (`172.16.x.x`) for all the tasks (alternatively in iDRAC see Hardware/Network Devices/NIC slot 2/Port 3).
On the bastion node (we won't reset IP for that) record the internal IP on the same iface (`172.16.41.244` in all the present configuration files).

```
for host in `curl -s http://wiki.rdu2.scalelab.redhat.com | html2text -b 0 | grep -e 'rvansa' | cut -f 2 -d '|'  | tr -d ' '`; do
    echo -ne "host $host {\n\thardware ethernet "
    sshpass -p 100yard- ssh root@${host}.rdu2.scalelab.redhat.com -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ip link show p2p3 2> /dev/null | tail -n 1 | sed 's/.*ether \([^ ]*\) .*/\1/' | tr -d '\n'
    echo -e ";\n\tfixed-address 172.16.0.x;\n}\n"
done
```

Copy all the files in this repository in the appropriate places, replacing the IP above with bastion node's IP. You can also replace the assignment name `cloud` by yours. Start the DNS and verify that it serves:

```
systemctl start named
dig @<local ip> api-int.<assignment>.scalelab
```

The above should be resolved bastion IP.

Update `/etc/dhcp/dhcpd.conf` to map MAC addresses to desired IPs (`172.16.0.254` for bootstrap node, `172.16.0.1-3` for masters and subsequent IPs for workers).

Download [rhcos-4.1.0-x86_64-installer-initramfs.img](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.1/4.1.0/rhcos-4.1.0-x86_64-installer-initramfs.img) and [rhcos-4.1.0-x86_64-installer-kernel](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.1/4.1.0/rhcos-4.1.0-x86_64-installer-kernel) and place these into `/var/lib/tftpboot/`.

```
cd /var/lib/tftpboot
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.1/4.1.0/rhcos-4.1.0-x86_64-installer-initramfs.img
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.1/4.1.0/rhcos-4.1.0-x86_64-installer-kernel
```

Start DHCP, HAProxy and TFTP (for PXE boot), and disable iptables which could interfere with those:
```
systemctl start dhcpd tftp haproxy
systemctl stop iptables
```

When verifying TFTP setup don't forget that firewall is likely to block this on client side, too.

At this point the HAProxy points to non-existent nodes so we cannot verify the HAProxy setup.

Bastion node will serve the ignition configs you've created before and RHCOS image. Create a new directory in /root/ and copy the `*.ign` files in there. Also download [rhcos-4.1.0-x86_64-metal-bios.raw.gz](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.1/latest/rhcos-4.1.0-x86_64-metal-bios.raw.gz) there.

You can either setup `httpd` to serve these files, or just start serving current directory on port 8000 through python (note that this will terminate as soon as you close the SSH session).

```
python -m SimpleHTTPServer &
```

## Install bootstrap node

Enter management interface (login `quads` / your-ticket-id = `123456`) and open console. Depending on iDRAC version it either opens new window or you need to use `javaws` from Oracle JRE. Then reboot the machine.

SCALE lab nodes boot from PXE by default, but I had mixed luck selecting them to boot from the correct NIC. Hitting PXE boot (F12) in the early screen and then selecting socket `Slot 2 IBA 40G Slot 4202 v1031` in the NIC menu worked best. Alternatively when you enter Grub command line and type `exit` it sometimes finds the correct TFTP.

If you're not lucky just map the virtual DVD drive to [rhcos-4.1.0-x86_64-installer.iso](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.1/4.1.0/rhcos-4.1.0-x86_64-installer.iso). Then, when you're prompted to start the installation add these parameters to the kernel (replace the IP and ignition config as needed):

```
coreos.inst=yes coreos.inst.install_dev=sda coreos.inst.image_url=http://172.16.41.244:8000/rhcos-4.1.0-x86_64-metal-bios.raw.gz coreos.inst.ignition_url=http://172.16.41.244:8000/bootstrap.ign
```

This step is avoided when you boot from PXE; good luck with fat fingers typing all those URLs. In any case make sure that the HTTP server is running on port 8000 on bastion node.

When the install completes, machine reboot and you get a login screen to RHCOS, it's a good time to ssh to bootstrap node (`ssh core@<bootstrap public IP> -i core_rsa`) and verify that the DHCP, DNS and HAProxy setup works. (Note: there are no credentials to login directly, you can close the iDRAC console now).
Here's a checklist:

1. `/etc/resolv.conf` contains just the bastion node IP (`172.16.x.x`)
2. `ifconfig` confirms that this node has IP `172.16.0.254` on the `p2p3` interface
3. `curl -k https://api-int.cloud.scalelab:22623/config/master` returns ugly long JSON

You can check `journalctl -e -u bootkube` but it should report the `etcd`s to be unreachable now.

## Install master nodes

Through iDRAC consoles install the 3 masters, ideally trough the PXE boot just selecting 'master' option. After reboot you should see the consoles looping a couple of times not being able to reach the `api-int.*:22623` address above and then briefly confirming that ignition configs have been applied; then you get the login screen (of no use).

If this for some reason does not work you can try enabling HAProxy access logs and confirm that the machine configs have been fetched: open `/etc/rsyslog.conf`, uncomment these lines:

```
$ModLoad imudp
$UDPServerRun 514
```
and add another line at the end:
```
local2.* /var/log/haproxy.log
```
Restart the rsyslog to be sure:
```
systemctl restart rsyslog
```

When master is correctly installed, you should see succesful confirmation in `journalctl -e -u bootkube` on the bootstrap node. After all masters are up, verify from your local machine that boostrap is completed:

```
./openshift-install wait-for bootstrap-complete --dir cloud --log-level debug
```

You can shutdown/repurpose the bootstrap node now as the log suggests.

Verify that you can contact the apiserver (from your local machine) and see the machines:

```
alias oc=`pwd`/oc --config `pwd`/cloud/auth/kubeconfig
oc whoami # should return system:admin
oc get nodes
```

## Install worker nodes

The installation follows the same process as with masters. Contrary to what the [documentation](https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html#installation-approve-csrs_installing-bare-metal) says, I could not see the workers as `NotReady` using `oc get node` command when these booted. However couple minutes after workers booted I could see some certificate signing requests as `Pending` in `oc get csr`. Let's approve them all:

```
oc adm certificate approve `oc get csr -o name`
```

A while after this finishes you should see them up in `oc get node`.

If you don't see the `csr`s, install worker again. Possibly once more. You'll get lucky, eventually.

It's possible to install further workers using the same approach even after the 24-hour certificate expiration limit the documentation mentions; that's probably limiting only the master bootstrap (make sure the bootstrap node is down when installing workers after the limit).

I haven't tested [adding RHEL workers](https://docs.openshift.com/container-platform/4.1/machine_management/adding-rhel-compute.html) yet.

## Get cluster operators running

Configure the storage for image-registry [as the docs says](https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html#installation-registry-storage-non-production_installing-bare-metal) (or `oc edit configs.imageregistry.operator.openshift.io cluster` and set the `spec.storage.emptyDir: {}` manually).

I had some other operators degraded (`oc get clusteroperator` or `oc get co`) after boot because some routes in `*.apps.cloud.scalelab` were not reachable (due to misconfiguration). The routers should be up and running on worker nodes and HAProxy should point to these machines. Check out what's happening there if you have trouble:

```
oc project openshift-ingress
oc get po
```

If all cluster operators are running the installation is complete; verify that from your local machine running

```
./openshift-install wait-for install-complete --dir cloud --log-level debug
```

## Istio

Maistra installed without issues in my case, following [install guide](https://maistra.io/docs/getting_started/install/). I have used the `base-install` control plane config, only setting `ior_enabled: true`. To my surprise wildcard routes are not supported in OCP 4.1 at all.

However when scaling the load I started having issues with `ingressgateway` running out of memory and crashing therefore; increase heap limits to 2GB.

Upstream can be installed based on the docs (don't forget to [do the platform setup](https://istio.io/docs/setup/kubernetes/platform-setup/openshift/) ) but the proxy-init containers don't work on RHCOS nodes; I had to replace the upstream ones with `docker.io/maistra/proxy-init-ubi8:0.12.0`. Since this only sets up firewalling rules it should not have an effect on benchmark (besides the intrinsic effect of running on RHCOS).

Also, for reproducible results it's better to disable the `horizontalpodautoscaler` (enabled by default in production installation) and scale `istio-ingressgateway` manually.

## Setting up client machines

Install necessary packages:
```
yum install -y java unzip
```

Add your DNS to `/etc/resolv.conf` and `chattr +i /etc/resolv.conf` (or use some cleaner way to prevent dhcpclient from updating it).



## Links

I've gathered some info regarding UPI installs during my efforts:

* https://github.com/christianh814/ocp4-upi-helpernode/blob/master/quickstart.md
* https://www.techbeatly.com/2019/07/openshift-4-libvirt-upi-installation.html/
* https://github.com/e-minguez/ocp4-upi-bm-pxeless-staticips/
    * Setting static IPs and DNS does not work for me, though; on masters & workers the ignition config provided during installation is mostly ignored, only the bootstrap api URL matters. (Could be worth replacing the URL when DHCP changes are not possible, but I don't know what else will break in later installation stages).
* https://access.redhat.com/solutions/4175151

## Virtualized master installation

First setup bridge network on the host OS:
```
# /etc/sysconfig/network-scripts/ifcfg-br0
DEVICE=br0
TYPE=Bridge
IPADDR=192.168.0.20
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
DNS=192.168.0.70
ONBOOT=yes
BOOTPROTO=static
DELAY=0
```
```
/etc/sysconfig/network-scripts/ifcfg-ens1f0np0
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=no
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=no
IPV6_DEFROUTE=no
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens1f0np0
UUID=cf60174f-4d21-4023-a9c5-f95c52cba601
DEVICE=ens1f0np0
ONBOOT=no
BRIDGE=br0
```

Run `ifdown ens1f0np0 && ifup br0 && ifup ens1f0np0` and make sure that you can ping the bridge IP from outside and ping other machines (e.g. the DNS) from host machine.

Then setup DNS entries, including reverse DNS lookup. In case you are mixing virtualized and non-virtualized nodes with multiple NICs make sure that the reverse lookup will point to the IP assigned to NIC which should host the intra-cluster traffic.

Add `option host-name "xxx"` to DHCP so that machines have their hostnames set up correctly (for multiple NICs, use this option everywhere). The virtualized nodes also need `option routers 192.168.0.xxx` to add default route; otherwise the nodes could not reach internet to download images. However the non-virtualized nodes which should probably reach internet through a pub interface and communicate using private interface *must not* have this option set.
If you're setting DHCP on the public interface consider using `deny unknown-clients` to not behave as a rogue DHCP server.

<del>
In order to reach Internet from the virtual machines we need to perform a NAT translation; assuming the host OS is RHEL8 with firewalld, start it and add a rule for each virtual machine:

```bash
firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=192.168.0.xxx masquerade'
firewall-cmd --reload
```
</del>

In order to let virtual nodes reach public internet we need to setup another interface (adding the NAT as above does not work anymore). For that we need to add second interface to the virtual nodes; we have to configure IPs, hostnames and disable default DNS setup: run `virsh net-edit default` and add this configuration:
```xml
<network>
  <name>default</name>
  <uuid><!-- do not change the UUID! --></uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:f3:02:f0'/>
  <dns enable='no'/> <!-- This is important -->
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
      <host mac='00:11:22:33:55:01' name='master1' ip='192.168.122.21'/>
      <host mac='00:11:22:33:55:02' name='master2' ip='192.168.122.22'/>
      <host mac='00:11:22:33:55:03' name='master3' ip='192.168.122.23'/>
    </dhcp>
  </ip>
</network>
```

To apply this configuration to the network run

```bash
virsh net-destroy default && virsh net-start default
```

When everything is set up you can install the virtual machines. Note that disk will be called `/dev/vda`; you can adjust parameters in PXE config or the interactive installer will ask you to reselect.

```
virt-install --virt-type=kvm --name=bootstrap --ram=32768 --vcpus=4 --pxe --network=bridge=br0,model=virtio,mac=06:11:11:11:11:00 --graphics vnc --disk path=/var/lib/libvirt/images/bootstrap.qcow2,size=20,bus=virtio,format=qcow2
```

This opens VNC on port `5900` (and subsequent for the other machines).

Note that the command above does not add the public-facing interface; to add it run

```bash
virsh attach-interface --domain ocp45-master1 --live --config --type network --source default --mac 00:11:22:33:55:01
```

Should you need to amend the network configuration with another host record, you can use

```bash
virsh net-update default add-last ip-dhcp-host '<host mac="00:11:22:33:55:04" ip="192.168.122.24" name="othernode1" />' --live --config
```

Note that it's possible to add that using `virsh net-edit default`, too, however changes will apply only after `virsh net-destroy default && virsh net-start default`. And there's one more caveat: It seems that the network interfaces won't be functional after network restart, you need to manually remove & add the interfaces:

```bash
virsh detach-interface ocp45-master1 network --live
virsh attach-interface --domain ocp45-master1 --live --config --type network --source default --mac 00:11:22:33:55:01
```