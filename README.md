# setup-slurm-and-nfs
https://hackmd.io/@stargazerwang/S1t60i6lF
# enable ssh root login
sudo vim /etc/ssh/sshd_config\
PermitRootLogin yes\
service sshd restart\
sudo passwd root

# generate ssh key
ssh-keygen -t rsa -b 4096(key length)\
vim .ssh/id_rsa.pub

# copy all rsa keys to .ssh/authorized_keys and then copy to all slaves
scp master:~/.ssh/authorized_keys .ssh/authorized_keys

# setup hosts
vim /etc/hosts\
copy all the hostname ip pairs together

# before installation of munge and slurm to ensure same UID and GID
groupadd -g 901 munge\
useradd -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -g munge -m -s /sbin/nologin -u 901 munge\
groupadd -g 902 slurm\
useradd -c "Slurm Workload Manager" -d /var/lib/slurm -g slurm -m -s /bin/bash -u 902 slurm

# install munge
apt-get install munge

# copy munge key to slaves
scp master:/etc/munge/munge.key /etc/munge/munge.key\
systemctl restart munge.service

# test if munge successfully
munge -n | ssh ubuntu1 unmunge
# if not check the time 
date
# check uid and gid 
id -u munge\
id -g munge

# install slurm
Master: apt-get install slurmctld\
Slave: apt-get install slurmd

# setup slurm config
https://slurm.schedmd.com/configurator.html\
(cluster name, control machines-slurmctldhost, compute machines-nodename-partitionname)\
slaves: slurmd -C\
(CPUs=1 Boards=1 SocketsPerBoard=1 CoresPerSocket=1 ThreadsPerCore=1 RealMemory=740)\
submit
# paste the config to /etc/slurm/slurm.conf to every node
vim /etc/slurm/slurm.conf
# add these two lines under slurmctldhost
AuthInfo=/var/run/munge/munge.socket.2\
AuthType=auth/munge

# create dir
find /var/log/munge /etc/munge -type d -execdir chmod 700 '{}' ';'\
find /var/run/munge -type d -execdir chmod 755 '{}' ';'\
find /var/log/slurm /var/spool/slurm -type d -execdir chmod 700 '{}' ';'\
find /var/run/slurm /etc/slurm -type d -execdir chmod 755 '{}' ';'\
chown -R munge:munge /var/log/munge /var/run/munge /etc/munge\
chown -R slurm:slurm /var/log/slurm /var/run/slurm /etc/slurm /var/spool/slurm
# create the dir that is not exist
mkdir /var/spool/slurm\
mkdir /var/run/slurm

# create /etc/slurm/cgroup.conf
### Slurm cgroup.conf Template

CgroupAutomount=yes\
#CgroupMountpoint=/sys/fs/cgroup\

ConstrainCores=no\
ConstrainRAMSpace=no

# test slurm
Master: slurmctld -D\
Slave: slurmd -D\
srun --nodes=1 --ntasks-per-node=1 bash -c "echo Hello world from \`hostname\`"

# start slurm
Master: systemctl start slurmctld.service\
Slave: systemctl start slurmd.service
# if state problem
scontrol update nodename=[nodename] state=resume


https://blog.devcloud.com.tw/ubuntu-nfs-install/
# nfs
sudo apt-get update
# Master
apt install nfs-kernel-server nfs-common\
mkdir /opt/nfs\
sudo vim /etc/exports\
/opt/nfs  192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)
# /opt/nfs 為要被掛載的目錄
# 192.168.0.0/24 : 允許 192.168.0.[0~255] 的網段中的使用該目錄進行掛載 (NFS Client)
systemctl restart nfs-kernel-server.service\
systemctl status nfs-kernel-server.service\
showmount -e localhost\
systemctl enable nfs-kernel-server (設定為開機啟動)

# Slave
apt-get install nfs-common\
showmount -e {NFS Server IP} # 請改為自己的 NFS Server IP\
mkdir /opt/nfs\
mount 192.168.0.50:/opt/nfs /opt/nfs\
df -h\
vim /etc/fstab
...
# 增加下列內容，並將 IP 修改為自己的 NFS Server IP.
192.168.0.50:/opt/nfs /opt/nfs nfs rw 0 0 

# ipoib, rdma
vi /etc/modules

ib_umad
ib_ipoib
xprtrdma
svcrdma

modprobe 上面四個
沒過就安裝 

ibstat
沒過就安裝

ip a
找到對的 port

編輯 /etc/netplan/<yaml>\
ex:\
    ibp4s0:\
      addresses:\
        - 172.16.0.206/24

ping master to check

# 新增 rdma port
apt install rdma-core\
apt install rdmacm-utils


/lib/systemd/system/nfs-server.service

[Unit]\
Description=NFS server and services\
DefaultDependencies=no\
Requires=network.target proc-fs-nfsd.mount\
Requires=nfs-mountd.service\
Wants=rpcbind.socket\
Wants=nfs-idmapd.service

After=local-fs.target\
After=network.target proc-fs-nfsd.mount rpcbind.socket nfs-mountd.service\
After=nfs-idmapd.service rpc-statd.service\
Before=rpc-statd-notify.service

GSS services dependencies and ordering\
Wants=auth-rpcgss-module.service\
After=rpc-gssd.service rpc-svcgssd.service

start/stop server before/after client\
Before=remote-fs-pre.target

Wants=nfs-config.service\
After=nfs-config.service

[Service]\
EnvironmentFile=-/run/sysconfig/nfs-utils

Type=oneshot\
RemainAfterExit=yes\
ExecStartPre=/sbin/modprobe xprtrdma\
ExecStartPre=/sbin/modprobe svcrdma\
ExecStartPre=/usr/sbin/exportfs -r\
ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS\
ExecStartPost=/bin/bash -c "sleep 3 && echo 'rdma 20049' | tee /proc/fs/nfsd/portlist"\
ExecStop=/usr/sbin/rpc.nfsd 0\
ExecStopPost=/usr/sbin/exportfs -au\
ExecStopPost=/usr/sbin/exportfs -f

ExecReload=/usr/sbin/exportfs -r

[Install]\
WantedBy=multi-user.target
