# setup_nimbus_cluster
Refer guide at: https://brettchapman.github.io/Nimbus_Cluster
and
https://github.com/TimoLassmann/dot-files/blob/master/additional/setup_nimbus_cluster.org
## 1. create multiple instances
https://nimbus.pawsey.org.au/project/instances/
Recommended: create all nodes in one go

## 2. setup ssh access in local machine
copy private key file to ~/.ssh/private_key.pem
add ssh key:
```bash
ssh-agent ## start ssh-agent
eval $(ssh-agent) ## print agent pid
ssh-add ~/.ssh/private_key.pem ## add ssh key
ssh-add -l ## list added keys
ssh -A ubuntu@ID ## log in to server, -A used to forward local ssh agent to server to login to other server
## without the -A option, you can also copy the key files to server and setup as local
## create ssh config file
# Sample SSH Client Configuration File

# Host-specific settings
StrictHostKeyChecking no
Host node-1
User ubuntu
Hostname 192.168.0.196
IdentityFile ~/.ssh/slnimbus.pem

Host node-2
User ubuntu
Hostname 192.168.0.152
IdentityFile ~/.ssh/slnimbus.pem

Host node-3
User ubuntu
Hostname 192.168.0.147
IdentityFile ~/.ssh/slnimbus.pem

Host node-4
User ubuntu
Hostname 192.168.0.74
IdentityFile ~/.ssh/slnimbus.pem

Host node-5
User ubuntu
Hostname 192.168.0.197
IdentityFile ~/.ssh/slnimbus.pem
```
## 3. add other nodes to /etc/hosts, to run pdsh commands on all nodes
add the nodes ID and name to /etc/hosts
```bash
# Your system has configured 'manage_etc_hosts' as True.
# As a result, if you wish for changes to this file to persist
# then you will need to either
# a.) make changes to the master file in /etc/cloud/templates/hosts.debian.tmpl
# b.) change or remove the value of 'manage_etc_hosts' in
#     /etc/cloud/cloud.cfg or cloud-config from user-data
#
127.0.1.1 node-1.novalocal node-1
127.0.0.1 localhost
192.168.0.196 node-1
192.168.0.152 node-2
192.168.0.147 node-3
192.168.0.74 node-4
192.168.0.197 node-5
# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```
## 4. install pdsh on all nodes
```bash
## before doing anything, update system packages on head node
sudo apt-get -yq update
sudo apt-get -yq upgrade
## install pdsh on head node
sudo apt-get install pdsh -yq ## -y: yes for all requests; -q: quiet
## update system and install pdsh on all nodes
pdsh -a sudo apt-get -yq update
pdsh -a sudo apt-get -yq upgrade
pdsh -a sudo apt-get install pdsh -yq
## in case of "pdsh@myhost: host2: rcmd: socket: Permission denied", do:
sudo echo "ssh" > /etc/pdsh/rcmd_default
```
## 5. install munge
```bash
sudo apt-get -yq install libmunge-dev libmunge2 munge ## head node
pdsh -a sudo apt-get -yq install libmunge-dev libmunge2 munge ## munge on all nodes

## distribute_munge.sh
sudo dd if=/dev/urandom of=~/munge.key bs=1c count=4M ## create "munge.key" binary file,requires root right
pdcp -a munge.key ~/munge.key ## copy key to all nodes
pdcp -a munge_per_node.sh ~/munge_per_node.sh ## copy munge_per_node.sh to all nodes
pdsh -a bash ./munge_per_node.sh ## set up all nodes

## munge_per_node.sh
sudo systemctl stop munge ## in case munge is running
sudo chown munge: munge.key
sudo mkdir -p /etc/munge/
sudo chown munge:munge /etc/munge/
sudo mv munge.key /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
sudo systemctl enable munge
sudo systemctl start munge

## check munge status in all nodes
pdsh -a systemctl status munge ## if master node in /etc/hosts, no need run separately on master node
```
## 6. install slurm
```bash
## prepare slurm.conf file
sudo apt install slurm-wlm-doc ## install slurm "configurator.html"

## download configurator.html to local machine
scp ubuntu@IP:/usr/share/doc/slurm-wlm/html/configurator.html ./

## open configurator.html in browser, and modify parameters:
ControlMachine = node-0 (if the master node is named as such)
NodeName = node-[1-3] (if named and numbered as such)
SlurmUser = slurm
StateSaveLocation = /var/spool/slurm-llnl
ProctrackType = Cgroup
TaskPlugin = Cgroup
AccountingStorageType = FileTxt
JobAcctGatherType = None
SlurmctldLogFile = /var/log/slurm-llnl/slurmctld.log
SlurmdLogFile = /var/log/slurm-llnl/slurmd.log
SlurmctldPidFile = /var/run/slurmctld.pid
SlurmdPidFile = /var/run/slurmd.pid
#### additional setting if using SlurmDBD
JobCompType = MySQL (set to none if leaving out SlurmDBD)
JobCompLoc = slurm_acct_db
JobCompHost = localhost
JobCompUser = slurm
JobCompPass = "password" (or whatever password was chosen for MariaDB)

## recommended to set password for all nodes to allow console login in case of ssh problem
## update_local_password.sh
yes password | sudo passwd ubuntu ## your password will be "password"
## set for all nodes
pdcp -a update_local_password.sh ~/
pdsh -a bash ./update_local_password.sh

```
