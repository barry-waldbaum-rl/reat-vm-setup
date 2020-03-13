# REAT VM Setup

Setup and test a base VM that runs:
- Docker network
- VNC container
- DNS container
- 6 Redis Labs node containers that appear to students as VMs.

Each student gets a VM with 6 RL nodes running in 2 clusters (north and south). Each node runs in a container but we make it look like a separate VM with its own IP. Each student gets a VNC desktop to their main VM where they can open browser sessions to admin consoles and SSH terminals to node shells and run rladmin.

The DNS and VNC containers need to be configured as follows:
- DNS - https://docs.google.com/document/d/1pDRZ8rHaR05UF4bU5SvwVkbM6FFj58apNsfxByibwoA/edit#heading=h.2gwmy0vc9jkp
- VNC - https://docs.google.com/document/d/1X8K2jZTwBLr_jG9a01-u_dxRrTwb2URNCy0zXoQZUG4/edit#

Build the following:
- Base VM
- Snapshot
- Image
- Test instance.

Steps:

1. Create a VPC with subnet 172.18.0.0/16 in the region where you want to run VMs.

Requirement | Specification
------------|--------------
Name | reat-vpc
Subnet Creation Mode | Custom
Subnet Name | reat-subnet
Subnet IP Address Range | 172.18.0.0/16

2. Create a firewall rule that allows ingress on all ports from all sources (0.0.0.0/0) to all targets.
 
3. Create the base VM in the region and VPC where you want to run instances.
  
Requirement  | Specification  
------------ | -------------
Name | reat-base-vm
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB
Networking | reat-vpc
  
4. SSH to the base VM from GCP console to finish setup.

5. Install 'vi', add 'trainee' user, put the user in the 'docker' group, and grant the user the ability to run 'sudo' (for RES install lab).

```bash 
# install and set default editor to vim
# then add 'trainee' user to 'docker' group so it can start and stop containers that run Redis software
sudo su
apt -y update
apt -y install vim

update-alternatives --config editor
```

Enter '3' to set the editor type.

```bash
adduser --disabled-password --gecos "" trainee
groupadd docker
usermod -aG docker trainee
```

Add the following line to /etc/sudoers using "sudo visudo" so 'trainee' user can run sudo

```bash
trainee ALL=(ALL) NOPASSWD:ALL
```

6. Install docker-ce.

```bash
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common 
```

and enter 'Y'.

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

apt-key fingerprint 0EBFCD88
   
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

apt-get update
apt-get -y install docker-ce
```

7. Switch to 'trainee' user to create the Docker network, DNS bind server, and add user scripts.

```bash
sudo su - trainee
```
8. Edit trainee's .bashrc file to uncomment the line '#force_color_prompt' so trainee user's prompt is green and distinguishable from VNC user (yellow) and RL node admins (red r white).

9. Create the Docker subnet and add the DNS bind server to it.

```bash
mkdir resolve
echo 'nameserver 172.18.0.20' > resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 rlabs

# for BIND DNS
docker run --name bind -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf -h ns.rlabs.org --net rlabs --restart=always -p 10000:10000/tcp --ip 172.18.0.20 rahimre/redislabs-training-bind

# for Coredns
docker pull coredns/coredns

# create Corefile and rlabs.db and put them in /home/trainee/coredns/

docker run --name coredns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf -h ns.rlabs.org --net rlabs --restart=always  -v /home/trainee/coredns/:/root/ --ip 172.18.0.20 coredns/coredns -conf /root/Corefile
```

10. Generate keys so students can 'silently' SSH from VNC container to base VM and RL nodes as if they were on their own machines. 

```bash
#
# reset
# rm -rf .ssh vnc_docker
#
mkdir .ssh
ssh-keygen -q -t rsa -N '' -f .ssh/id_rsa 2>/dev/null <<< y >/dev/null
cp -r .ssh/id_rsa.pub .ssh/authorized_keys 

mkdir vnc_docker
cp -r .ssh/ vnc_docker/ssh 
```

11. Create Dockerfile that will be used to build the VNC Docker image on the base VM (you only run VNC container in student instance startup scripts because the container needs to receive the instance's private IP to append to SSH alias commands).

```bash
cat << EOF > vnc_docker/Dockerfile
## Custom Dockerfile
FROM consol/ubuntu-xfce-vnc

# Switch to root user to install additional software
USER 0

## Install SSH, DNS Utils, and the VNC user's bashrc
RUN apt update; apt install -y ssh dnsutils;
RUN mkdir /headless/.ssh
COPY ./ssh /headless/.ssh
RUN chown -R 1000 /headless/.ssh/
COPY bashrc /headless/.bashrc
RUN chown -R 1000 /headless/.bashrc
COPY resolve/resolv.conf /etc/resolv.conf

## switch back to default user
USER 1000
EOF
```

12. Create the bashrc and scripts for students to start, stop, and SSH to RL nodes as if they were on machines instead of containers as well as run and remove a Redis server container for running redis-server and redis-cli in lab 2.

NOTE: VNC user prompt will be set to yellow to distinguish it from VM user 'trainee' (green) and RL node admins (red or white).

```bash
cat << EOF > vnc_docker/bashrc
source \$STARTUPDIR/generate_container_user

export PS1='\e[1;33m\u@\h\e[m:\e[1;34m\w\e[m\$ '

alias ssh_installer="ssh trainee@\$INT_IP"

alias start_redis='ssh -t trainee@\$INT_IP docker run -it --name redis -h redis -w / redis bash'
alias stop_redis='ssh -t trainee@\$INT_IP docker container rm \$\(docker container ls -q -f '\''status=exited'\''\)'

alias run_dnsutils="ssh -t trainee@\$INT_IP ./scripts/run_dnsutils.sh "

alias ssh_n1="ssh -t trainee@\$INT_IP docker exec -it n1 bash "
alias ssh_n2="ssh -t trainee@\$INT_IP docker exec -it n2 bash "
alias ssh_n3="ssh -t trainee@\$INT_IP docker exec -it n3 bash "
alias ssh_s1="ssh -t trainee@\$INT_IP docker exec -it s1 bash "
alias ssh_s2="ssh -t trainee@\$INT_IP docker exec -it s2 bash "
alias ssh_s3="ssh -t trainee@\$INT_IP docker exec -it s3 bash "

alias start_n1="ssh -t trainee@\$INT_IP docker start n1 "
alias start_n2="ssh -t trainee@\$INT_IP docker start n2 "
alias start_n3="ssh -t trainee@\$INT_IP docker start n3 "

alias stop_n1="ssh -t trainee@\$INT_IP docker stop n1 "
alias stop_n2="ssh -t trainee@\$INT_IP docker stop n2 "
alias stop_n3="ssh -t trainee@\$INT_IP docker stop n3 "

alias start_s1="ssh -t trainee@\$INT_IP docker start s1 "
alias start_s2="ssh -t trainee@\$INT_IP docker start s2 "
alias start_s3="ssh -t trainee@\$INT_IP docker start s3 "

alias stop_s1="ssh -t trainee@\$INT_IP docker stop s1 "
alias stop_s2="ssh -t trainee@\$INT_IP docker stop s2 "
alias stop_s3="ssh -t trainee@\$INT_IP docker stop s3 "

alias start_north_nodes="ssh -t trainee@\$INT_IP ./scripts/start_north_nodes.sh "
alias start_south_nodes="ssh -t trainee@\$INT_IP ./scripts/start_south_nodes.sh "

alias create_north_cluster="ssh -t trainee@\$INT_IP ./scripts/create_north_cluster.sh "
alias create_south_cluster="ssh -t trainee@\$INT_IP ./scripts/create_south_cluster.sh "

EOF

mkdir scripts

cat << EOF > scripts/start_north_nodes.sh
docker kill n1; docker rm n1;
docker kill n2; docker rm n2;
docker kill n3; docker rm n3;
docker run -d --cap-add=ALL --name n1 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n1.rlabs.org --net rlabs --ip 172.18.0.21 redislabs/redis
docker run -d --cap-add=ALL --name n2 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n2.rlabs.org --net rlabs --ip 172.18.0.22 redislabs/redis
docker run -d --cap-add=ALL --name n3 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n3.rlabs.org --net rlabs --ip 172.18.0.23 redislabs/redis
docker exec --user root n1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sleep 60
EOF

cat << EOF > scripts/start_south_nodes.sh
docker kill s1; docker rm s1;
docker kill s2; docker rm s2;
docker kill s3; docker rm s3;
docker run -d --cap-add=ALL --name s1 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s1.rlabs.org --net rlabs --ip 172.18.0.31 redislabs/redis
docker run -d --cap-add=ALL --name s2 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s2.rlabs.org --net rlabs --ip 172.18.0.32 redislabs/redis
docker run -d --cap-add=ALL --name s3 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s3.rlabs.org --net rlabs --ip 172.18.0.33 redislabs/redis
docker exec --user root s1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sleep 60
EOF

cat << EOF > scripts/create_north_cluster.sh
docker exec -it n1 bash -c "/opt/redislabs/bin/rladmin cluster create persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.21 \
        name north.rlabs.org username admin@rlabs.org password admin";

docker exec -it n2 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.22 \
        username admin@rlabs.org password admin nodes 172.18.0.21";

docker exec -it n3 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.23 \
        username admin@rlabs.org password admin nodes 172.18.0.21";
EOF

cat << EOF > scripts/create_south_cluster.sh
docker exec -it s1 bash -c "/opt/redislabs/bin/rladmin cluster create persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.31 \
        name south.rlabs.org username admin@rlabs.org password admin";
docker exec -it s2 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.32 \
        username admin@rlabs.org password admin nodes 172.18.0.31";
docker exec -it s3 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.33 \
        username admin@rlabs.org password admin nodes 172.18.0.31";
EOF

cat << EOF > scripts/run_dnsutils.sh
docker kill dnsutils; docker rm dnsutils
docker run --name dnsutils -it -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --net rlabs --hostname dnsutils.rlabs.org --ip 172.18.0.6 tutum/dnsutils
EOF

# make the scripts executable
chmod 755 scripts/start_north_nodes.sh
chmod 755 scripts/start_south_nodes.sh
chmod 755 scripts/create_north_cluster.sh
chmod 755 scripts/create_south_cluster.sh
chmod 755 scripts/run_dnsutils.sh
```

13. Run scripts to start Redis Labs nodes running in their containers (3 north, 3 south). You run these on the base VM so students don't have to download Docker images in class (that could overload the network).

```bash
scripts/start_north_nodes.sh
scripts/start_south_nodes.sh
```

14. Run scripts to build Redis Labs clusters from their nodes.

```bash
scripts/create_north_cluster.sh
```

```bash
scripts/create_south_cluster.sh
```

15. Run the Redis Insight utility app as a container so students can view database contents in a GUI.

```bash
docker run -d --name insight -v redisinsight:/db -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always  --hostname insight.rlabs.org --net rlabs --ip 172.18.0.4  redislabs/redisinsight
```

16. Build the Docker image for the VNC container. You wait to run VNC containers when you create student instances because they pass in the instance IP. 

```bash
cd vnc_docker
docker build -t re-vnc .
```

You're finished creating the base VM.

17. Create a snapshot from the VM called 'reat-snap'.

18. Create an image from the snapshot called 'reat-image'.

19. Create a test instance from the image defined by the specs below which include a startup script.

NOTE: Be sure to add the startup script below which runs VNC container on start up and passes in the instance's internal IP address so, once signed in, a VNC user can SSH back to the main VM as user 'trainee' (for installing Redis Enterprise Software in one of the labs) as well as SSH to RL nodes (n1-n3 and s1-s3) as an RL admin and run 'rlaadmin'.

```bash
docker run -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'` -p 80:6901 -e VNC_PW=trainee! --net rlabs --ip 172.18.0.2  --name controller -h controller.rlabs.org -d re-vnc;
```

Requirement  | Specification  
------------ | -------------
Name | singlenode-01
CPU | 4
Memory | 15 GB
Boot disk | reat-image
Disk size | 30 GB
Network | reat-vpc
Startup script | see above

20. Point your laptop browser to the VM's public IP on port 80. You can get the public IP from GCP admin console.

21. Sign in to VNC desktop with password 'trainee!'.

22. In VNC desktop, open Chrome browser and point it to RL admin consoles on port 8443. You can use either hostnames or IPs.

```bash
n1 = 172.18.0.21
n2 = 172.18.0.22
n3 = 172.18.0.23

s1 = 172.18.0.31
s2 = 172.18.0.32
s3 = 172.18.0.33
```

23. Open Applications > Terminal (top-left) and run commands to restart RL nodes, create clusters, SSH to node VMs (containers really), or SSH to the main VM for the Software Installation lab.

```bash
# restart RL nodes
start_north_nodes
start_south_nodes

# create RL clusters
create_north_cluster
create_south_cluster

# SSH to RL nodes so you can run rladmin
ssh_n1
rlcheck
rladmin status
redis-cli -p 12000 -h redis-12000.north.rlabs.org
exit
exit
ssh_n2
exit
ssh_n3
exit

# start, test, and stop OSS Redis container
start_redis
> redis-server &
> redis-cli
127.0.0.1:6379> keys *
(empty list)
127.0.0.1:6379> exit
exit
stop_redis

# SSH to the main VM as user 'trainee' with sudo/root permissions
ssh_installer
sudo docker images
sudo docker ps
sudo docker network list
sudo docker network inspect rlabs
sudo docker image inspect redislabs/redis
exit
```

If you want to add dnsutils to test DNS on the rlabs network run
```bash
run_dnsutils
```

