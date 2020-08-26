# JetSki install in the lab
## clone Jetski
The JetSki project is hosted on github @ https://github.com/redhat-performance/JetSki
```
git clone https://github.com/redhat-performance/JetSki.git
cd JetSki
```

## check allocation
Check the [scalelab wiki](http://wiki.rdu2.scalelab.redhat.com/) to make sure the allocation is ready. 
*DO NOT* run JetSki until the allocation is ready (trust me, it makes things worse)
Check the OCPINV link from the Summary table in the wiki
remember the `pm_user` , `pm_password` and the `pm_addr` of the first node in `nodes`

## update JetSki configuration
Change `ansible-ipi-install/group_vars/all.yml`
```yaml
cloud_name: cloudN #the cloud name
ansible_ssh_pass: password #set to the user password for maching login 
ansible_ssh_key: core_rsa_lta #change to the location of the shared key for the lab
version: 4.5.4 #whatever version we are testing
pullsecret: '...' #add your pull secret, be sure to keep it in single quotes (')
foreman_url: formeman #get the url from http://pastebin.test.redhat.com/890421
worker_cound: 12 # set the worker cound to the number of allocated machines - 4 (bastion + 3 masters) or 0
```
Also change `ansible-ipi-install/inventory/jetski/hosts`

```
domain="scalelab"
cluster="ocp45"
cluster_random=false
```

## Run JetSki
```
cd ansible-ipi-install
ansible-playbook -i inventory/jetski/hosts playbook-jetski.yml
```
If your allocation has older iDRAC (version < 4.20.20.20) then this install can take a few hours

If the install worked (no Errors from ansible output) then the cluster is created and the first node 

## Post Install steps

###Log into the bastion 
The bastion is the `pm_addr` hostname without the `mgmt-` prefix from the first node in `nodes`.
For example, host.redhat.com not mgmt-host.redhat.com
```
ssh kni@host.redhat.com
```
If you get rust errors you probably need to add the `core_rsa_lta` from `all.yml` into your ssh config by copying it to `~/.ssh/` and make sure it only has user read/write access (600/-rw-------.)

Verify oc works
```
oc whoami #system:admin
oc get nodes -o wide #expect to see all workers and masters
oc -n openshift-console get route/console -o jsonpath='{.spec.host}' #note the FQDN after apps
```
The bastion may be running haproxy already (I forget) but either way we are going to add it to forward external requests to the correct hosts.
```
sudo yum install -y haproxy
sudo vi /etc/haproxy/haproxy.cfg
```
Edit `/etc/haproxy/haproxy.cfg`
```
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
defaults
    mode                    tcp
    log                     global
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 30000
      frontend stats
          bind *:8404
          mode http
          stats enable
          stats uri /stats
          stats refresh 10s
          stats admin if TRUE
      frontend api
          bind *:6443
          option tcplog
          default_backend api
      frontend machineconfig
          bind *:22623
          option tcplog
          default_backend machineconfig
      frontend router_http
          bind *:80
          mode http
          option httplog
          option http-server-close
          option forwardfor       except 127.0.0.0/8
          default_backend router_http
      frontend router_https
          bind *:443
          option tcplog
          default_backend router_https
      backend api
          balance     roundrobin
          server  master-0 192.168.222.10:6443 check
          server  master-1 192.168.222.11:6443 check
          server  master-2 192.168.222.12:6443 check
      backend machineconfig
          balance     roundrobin
          server  master-0 192.168.222.10:22623 check
          server  master-1 192.168.222.11:22623 check
          server  master-2 192.168.222.12:22623 check
      backend router_http
          mode http
          balance     roundrobin
          server worker000 192.168.222.13:80 check
          server worker001 192.168.222.14:80 check
          server worker002 192.168.222.15:80 check
          server worker003 192.168.222.16:80 check
          server worker004 192.168.222.17:80 check
          server worker005 192.168.222.18:80 check
          server worker006 192.168.222.19:80 check
          server worker007 192.168.222.20:80 check
          server worker008 192.168.222.21:80 check
          server worker009 192.168.222.22:80 check
          server worker010 192.168.222.23:80 check
          server worker011 192.168.222.24:80 check
      backend router_https
          balance     roundrobin
          server worker000 192.168.222.13:443 check
          server worker001 192.168.222.14:443 check
          server worker002 192.168.222.15:443 check
          server worker003 192.168.222.16:443 check
          server worker004 192.168.222.17:443 check
          server worker005 192.168.222.18:443 check
          server worker006 192.168.222.19:443 check
          server worker007 192.168.222.20:443 check
          server worker008 192.168.222.21:443 check
          server worker009 192.168.222.22:443 check
          server worker010 192.168.222.23:443 check
          server worker011 192.168.222.24:443 check
```
Double check the IPs from the `oc get nodes -o wide` match the IPs from the sample haproxy.cfg and the worker counts are the same.

Restart HAProxy
```
sudo systemctl start haproxy # incase it isn't running
sudo systemctl restart haproxy
```

Update dnsmasq
```
ip a #
sudo vi /etc/dnsmasq.conf
```
Edit `/etc/dnsmasq.conf` by adding the following section
```
# ocp45
address=/.apps.ocp45.scalelab/192.168.222.1
address=/api.ocp45.scalelab/192.168.222.1
address=/api-int.ocp45.scalelab/192.168.222.1
address=/etcd-0/192.168.222.10
address=/etcd-1/192.168.222.11
address=/etcd-2/192.168.222.12
address=/_etcd-server-ssl._tcp/192.168.222.10
```
The FQDN `ocp45.scalelab` probably does not match the FQDN from `oc -n openshift-console get route/console -o jsonpath='{.spec.host}'` above so update the dnsmasq.conf to match the `oc` output.

Double check the IP addresses from `ip a` is 192.168.222.1 otherwise replace that address with the bastion's internal IP address. Also double check the IP addres for the 3 master nodes.

Restart dnsmasq
```
sudo systemctl start dnsmasq
sudo systemctl restart dnsmasq
```

### Create a ocp admin user
```
yum install -y httpd-tools #provides htpasswd
htpasswd -c -B -b ./user.htpasswd username password #please change the username and password
oc create secret generic htpass-secret --from-file=htpasswd=./user.htpasswd -n openshift-config
cat <<EOF | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
EOF
oc adm policy add-cluster-role-to-user cluster-admin username #ignore warning about 'username' not found
oc login api.ocp45.scalelab:6443 -u username -p password
```
The FQDN may not match what you got before so update the login command 

### Update image-registry
The Openshift image registry is created without persistence. We can set the persistence to an empty dir
```
oc patch configs.imageregistry/cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
oc -n openshift-image-registry delete pods --all
oc -n openshift-image-registry delete job --all
```
## Update Laptop
If you want to reach the cluster from your laptop then you need to update NetworkManager to use dnsmaq.

`sudo vi /etc/NetworkManager/NetworkManager.conf`
```
dns=dnsmasq
```
Then `sudo vi /etc/NetworkManager/dnsmasq.d/scalelab.conf`
```
# Scalelab
address=/api.ocp45.scalelab/10.1.44.98
address=/api-int.ocp45.scalelab/10.1.44.98
address=/.apps.ocp45.scalelab/10.1.44.98
```
