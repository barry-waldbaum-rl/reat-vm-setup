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
- Base VM, snapshot, and image with no DNS server
- Base VM, snapshot, and image with DNS
- Test instance with VNC.

Steps:

1. Create a VPC with subnet 172.18.0.0/16 in the region where you want to run VMs.

Requirement | Specification
------------|--------------
Name | rat-vpc
Subnet Creation Mode | Custom
Subnet Name | rat-subnet
Subnet IP Address Range | 172.18.0.0/16

2. Create a firewall rule that allows ingress on all ports from all sources (0.0.0.0/0) to all targets.
 
3. Create the base VM in the region and VPC where you want to run instances.
  
Requirement  | Specification  
------------ | -------------
Name | rat-no-dns
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB
Networking | reat-vpc
  
4. SSH to the base VM from GCP console to finish setup.

5. Install 'vi', add user 'trainee', add the user to the 'docker' group, and give the student permissions to run Docker commands without entering 'sudo'.

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

Add the following line to /etc/sudoers using "sudo visudo" so students can start and stop containers without entering 'sudo'.

```bash
trainee ALL=(ALL) NOPASSWD:ALL
```

6. Install Docker.

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

7. Switch to 'trainee' user to create the Docker network, add user scripts, and build the VNC Docker image.

```bash
sudo su - trainee


```

8. Edit trainee's .bashrc file to uncomment the line '#force_color_prompt' so 'trainee' user shell prompt on the 'controller' VM is green. Shell prompt color for 'default' user on the VNC container is yellow. Shell prompt color for 'root' users on RL nodes is white.

9. Generate keys so students can 'silently' SSH from VNC container to base VM and RL nodes as if they were on their own machines. 

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

10. Create bashrc with aliases, and a set of scripts for students so they can:
- Start, stop, and SSH to RL nodes as if they were on machines instead of containers
- Run a Redis server in a container for lab 2 that can be discarded when done
- Run DNS Utils in a container to diagnose DNS on the Docker network.

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

11. Create the Docker network so you can run RL nodes, Redis Insight, and an internal DNS server on a private network.

```bash
mkdir resolve
echo 'nameserver 172.18.0.20' > resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 rlabs


```

12. Build a Docker image for the VNC container. The container runs from the VM startup script when you create student instances. It receives the VM's internal IP for aliases so students can 'silently' SSH from VNC container to base VM and RL nodes as if they were on their own machines. 

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

## switch back to default user
USER 1000
EOF

cd vnc_docker
docker build -t re-vnc .


```

13. Run scripts to start Redis Labs nodes running in their containers (3 north, 3 south). You run these on the base VM so students don't have to download Docker images in class (that could overload the network).

```bash
cd ..
scripts/start_north_nodes.sh
scripts/start_south_nodes.sh


```

14. Run scripts to build Redis Labs clusters from their nodes.

```bash
scripts/create_north_cluster.sh
scripts/create_south_cluster.sh


```

15. Run the Redis Insight utility app as a container so students can view database contents in a GUI.

```bash
docker run -d --name insight -v redisinsight:/db -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always  --hostname insight.rlabs.org --net rlabs --ip 172.18.0.4  redislabs/redisinsight


```

You're finished creating the base VM minus a DNS server. It's time to save your work in a GCP image so you don't have to do these steps again.

16. Create a snapshot from the VM called 'rat-no-dns'.

17. Create an image from the snapshot called 'rat-no-dns'.

Now it's time to configure the DNS server and test your configuration. This may change over time so it's good to start with a base image with most of the work already done and saved.

18. Create a new instance called 'rat-with-dns' from the saved image.

19. SSH to the instance from GCP console.

20. Switch to user trainee.

```bash
sudo su - trainee


```

21. Add a DNS server to the Docker network so hostnames get resolved in the private network.

You will have two options: BIND and CoreDNS. Only BIND works right now. CoreDNS doesn't resolve cluster names yet.

BIND DNS

```bash
docker run --name bind -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --net ralbs --hostname ns.rlabs.org --ip 172.18.0.20 -p 10000:10000/tcp sameersbn/bind


```

CoreDNS - Create Corefile and rlabs.db, put them in /home/trainee/coredns/, and run:

```bash
docker run --name coredns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf -h ns.rlabs.org --net rlabs --restart=always  -v /home/trainee/coredns/:/root/ --ip 172.18.0.20 coredns/coredns -conf /root/Corefile


```

22. If using BIND DNS:
- Get the VMs public IP from GCP console
- Point a laptop browser to https://<public-ip>:10000
- Sign in as 'root' with 'password'
- Configure the server according to steps here:
https://docs.google.com/document/d/1pDRZ8rHaR05UF4bU5SvwVkbM6FFj58apNsfxByibwoA/edit#heading=h.2gwmy0vc9jkp
 
23. Run this script to start a container with DNS Utils in it so you can test your DNS config.

```bash
./scripts/run_dnsutils.sh


```

24. Run the following commands to make sure DNS is working

```bash
nslookup n1.rlabs.org
nslookup s1.rlabs.org
dig @ns.rlabs.org north.rlabs.org
dig @ns.rlabs.org south.rlabs.org
```
 
25. Create a snapshot from the VM called 'rat-with-dns'.

26. Create an image from the snapshot called 'rat-with-dns'.

27. Create a test or student instance from the image as follows:

Requirement  | Specification  
------------ | -------------
Name | rat-controller-01
CPU | 4
Memory | 15 GB
Boot disk | rat-with-dns
Disk size | 30 GB
Network | rat-vpc
Startup script | see below

START UP SCRIPT: Runs VNC in a container and exposes it on port 80 so students can access admin consoles without being firewall blocked on port 8443.

There seem to be two different sets of 'awk' commands that work on GCP instances these days.

Here's the original script with one 'awk' command that still works.

```bash
docker run -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'` -p 80:6901 -e VNC_PW=trainee! --net rlabs --ip 172.18.0.2  --name controller -h controller.rlabs.org -d re-vnc
```

Here's the newer one with two 'awk' commands that works on newer and larger VMs.

```bash
docker run -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F : '{ print $2 }' | awk -F ' ' '{ print $1}'` -p 80:6901 -e VNC_PW=trainee! --net rlabs --ip 172.18.0.2  --name controller -h controller.rlabs.org -d re-vnc
```

28. Point your laptop browser to the VM's public IP on port 80. You can get the public IP from GCP admin console.

29. Sign in to VNC desktop with password 'trainee!'.

30. In VNC desktop, open Chrome browser and point it to RL admin consoles on port 8443. You can use either hostnames or IPs.

```bash
n1 = 172.18.0.21
n2 = 172.18.0.22
n3 = 172.18.0.23

s1 = 172.18.0.31
s2 = 172.18.0.32
s3 = 172.18.0.33
```

31. Open Applications > Terminal (top-left) and run commands to restart RL nodes, create clusters, SSH to node VMs (containers really), or SSH to the main VM for the Software Installation lab.

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

