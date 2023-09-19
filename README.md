# setup_nimbus_cluster
slight modification from guides at https://brettchapman.github.io/Nimbus_Cluster
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

## with this, can ssh to each node simply:
ssh host_name
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
#### use lscpu and free -m to get infomation
ControlMachine = node-1
NodeName = node-[1-3] ## include master node as compute node as well
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

## copy hosts to each node
pdcp -a /etc/hosts ~/
pdsh -a sudo mv ~/hosts /etc/hosts

## install slurm on each node
### setup_host_for_slurm.sh
sudo apt-get install -yq slurm-client

sudo mkdir -p /etc/slurm-llnl
sudo mkdir -p /var/spool/slurm-llnl
sudo mkdir -p /var/spool/slurmd
sudo mkdir -p /var/log/slurm-llnl

sudo -- sh -c "cat > /etc/slurm-llnl/cgroup.conf << 'EOF'
CgroupAutomount=yes
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=yes"

sudo mv ~/slurm.conf /etc/slurm-llnl/slurm.conf
sudo chown slurm: /etc/slurm-llnl/slurm.conf
sudo chown slurm: /var/spool/slurm-llnl
sudo chown slurm: /var/spool/slurmd
sudo chown slurm: /var/log/slurm-llnl
sudo chown slurm: /etc/slurm-llnl/cgroup.conf

### install_slurm.sh
sudo apt-get install -yq slurmctld slurmdbd
pdsh -a sudo apt-get -yq update
pdsh -a sudo apt-get -yq upgrade
pdsh -a sudo apt-get -yq install slurmd pdsh
pdcp -a slurm.conf ~/slurm.conf
pdcp -a setup_host_for_slurm.sh ~/setup_host_for_slurm.sh
pdsh -a bash ./setup_host_for_slurm.sh
sudo mkdir -p /etc/slurm-llnl
sudo mv ~/slurm.conf /etc/slurm-llnl/slurm.conf
sudo chown slurm: /etc/slurm-llnl/slurm.conf
sudo mkdir -p /var/spool/slurm-llnl
sudo chown slurm: /var/spool/slurm-llnl
sudo mkdir -p /var/spool/slurmd
sudo chown slurm: /var/spool/slurmd
sudo mkdir -p /var/log/slurm-llnl
sudo chown slurm: /var/log/slurm-llnl
sudo touch /var/log/slurm_jobacct.log
sudo chown slurm: /var/log/slurm_jobacct.log

sudo -- sh -c "cat > /etc/slurm-llnl/cgroup.conf << 'EOF'
CgroupAutomount=yes
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=yes"

sudo chown slurm: /etc/slurm-llnl/cgroup.conf
##
```
## 7. install MariaDB and MySQL
```bash
## enable slurmbd
sudo systemctl enable slurmdbd

## Install, start and enable MariaDB:
sudo apt-get install mariadb-server -yq
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo systemctl status mariadb

## configure MariaDB root password
sudo /usr/bin/mysql_secure_installation ## Set password as “password” (as selected in the slurm.conf flle) and select Y for remaining questions.

## grant permission and create mariaDB database
sudo mysql -p (enter chosen password)
MariaDB> grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'password' with grant option;
MariaDB> SHOW VARIABLES LIKE 'have_innodb';
MariaDB> create database slurm_acct_db;
MariaDB> show grants;
MariaDB> quit;

## edit mariaDB config file
sudo vim /etc/mysql/my.cnf
## append following:
[mysqld]
innodb_buffer_pool_size=16G
innodb_log_file_size=64M
innodb_lock_wait_timeout=900

## in case of vim encryption key, to remove the key:
:set key= ## followed by :w to save

## implement changes
sudo systemctl stop mariadb
sudo mv /var/lib/mysql/ib_logfile? /tmp/
sudo systemctl start mariadb
sudo mysql -p
MariaDB> SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
MariaDB> quit;
```
## 8. slurmdbd setup
```bash
zcat /usr/share/doc/slurmdbd/examples/slurmdbd.conf.simple.gz > slurmdbd.conf

## Edit the slurmdbd.conf file (sudo vim slurmdbd.conf) set the following:
Set DbdHost to localhost
Set StorageHost to localhost
Set StorageLoc to slurm_acct_db
Set StoragePass to password (or whatever password was chosen to access MariaDB)
Set StorageType to accounting_storage/mysql
Set StorageUser to slurm
Set LogFile to /var/log/slurm-llnl/slurmdbd.log
Set PidFile to /var/run/slurmdbd.pid
Set SlurmUser to slurm

## Move the slurmdbd.conf to the slurm directory:
sudo mv slurmdbd.conf /etc/slurm-llnl/
sudo chown slurm:slurm /etc/slurm-llnl/slurmdbd.conf
sudo chmod 664 /etc/slurm-llnl/slurmdbd.conf

## Run SlurmDBD interactively with debug options to check for any errors 
sudo slurmdbd -D -vvv ## ctrl+c to end

## Check that the “slurm_acct_db” has been populated with tables:
mysql -p -u slurm slurm_acct_db
MariaDB> show tables;
MariaDB> quit;

## start slurmdbd service
sudo systemctl enable slurmdbd
sudo systemctl start slurmdbd
sudo systemctl status slurmdbd
```
## 9. start slurmdbd, slurmctld, slurmd on all nodes
```bash
## start and status check
sudo systemctl enable slurmdbd
sudo systemctl start slurmdbd
sudo systemctl status slurmdbd

sudo systemctl stop slurmctld
sudo systemctl start slurmctld
sudo systemctl status slurmctld

pdsh -a sudo systemctl stop slurmd
pdsh -a sudo systemctl enable slurmd
pdsh -a sudo systemctl status slurmd

pdsh -a sudo slurmd
pdsh -a sudo slurmd -Dcvvv
## run check
sinfo (displays node information)
sudo sacct (requires SlurmDBD and shows previous or running jobs)
scontrol show jobs (shows details of currently running jobs)
scontrol ping (pings slurmctld and shows its status)

## if some service inactive, try:
sudo service slurmdbd restart
sudo service slurmctld restart
pdsh -a sudo service slurmd restart
sudo systemctl status slurmdbd
sudo systemctl status slurmctld
pdsh -a sudo systemctl status slurmd
```
## 10. mount shared /data volume to all nodes
```bash
## setup_nfs.sh
pdsh -a sudo apt-get -yq install nfs-common
sudo apt-get install -yq rpcbind nfs-kernel-server

sudo -- sh -c "cat > /etc/exports << 'EOF'
/data 192.168.0.152(rw,sync,no_subtree_check)
/data 192.168.0.147(rw,sync,no_subtree_check)
/data 192.168.0.74(rw,sync,no_subtree_check)
/data 192.168.0.197(rw,sync,no_subtree_check)"

sudo /etc/init.d/rpcbind restart
sudo /etc/init.d/nfs-kernel-server restart
sudo exportfs -r

pdsh -a sudo mkdir /data
pdsh -a sudo chown -R ubuntu:ubuntu /data
pdsh -a sudo mount 192.168.0.196:/data /data

pdsh -a ls /data
sudo chown -R ubuntu:ubuntu /data
## RUN:
bash ./setup_nfs.sh
```
## Now, you should be able to submit jobs to all nodes!!!
```bash
## to use conda (installed at /data/tools/minoconda3) on all nodes:
source /data/tools/miniconda3/bin/activate nf-env
## add mybin to PATH
export PATH=$PATH:/data/tools/mybin
pdcp -a .bashrc ~/.bashrc ## copy bash config file to all nodes
```

