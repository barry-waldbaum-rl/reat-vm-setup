# REAT VM Setup

Setup and test a VM running the following:
- Docker network
- VNC server
- DNS bind server
- Redis Labs nodes that appear to run on VMs.

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
  
4. SSH to the base VM from GCP to finish setup.

5. Install 'vi', add 'trainee' user, put the user in the 'docker' group, and grant the user the ability to run 'sudo' (for RES install lab).

```bash 
sudo su
apt -y update
apt -y install vim

# Set the default editor to vim
update-alternatives --config editor

# Add 'trainee' user to 'docker' group so it can start and stop containers that run Redis software
adduser --disabled-password --gecos "" trainee
groupadd docker
usermod -aG docker trainee

# add the following line to /etc/sudoers using "sudo visudo" so 'trainee' user can run sudo
trainee ALL=(ALL) NOPASSWD:ALL
```

6. Install docker-ce.

```bash
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
    
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

7. Create the Docker subnet and add the DNS bind server to it.

```bash
mkdir resolve
echo 'nameserver 172.18.0.20' > resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 redislabs

docker run --name bind -d -v ~/resolve/resolv.conf:/etc/resolv.conf  --net redislabs --restart=always -p 10000:10000/tcp --ip 172.18.0.20 rahimre/redislabs-training-bind
```

8. Create keys so students can silently SSH from the VNC container to the base VM and run commands on RL nodes as if they were running on machines or VMs instead of containers. 

```bash
#
# reset
# rm -rf .ssh vnc_docker
#
ssh-keygen -q -t rsa -N '' -f .ssh/id_rsa 2>/dev/null <<< y >/dev/null
cp -r .ssh/id_rsa.pub .ssh/authorized_keys 

mkdir vnc_docker
cp -r .ssh/ vnc_docker/ssh 
```

9. Create the Dockerfile that will build the VNC Docker image.

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
```

10. Build the VNC Docker image.

```bash
cd vnc_docker
docker build -t re-vnc .
```

11. Create the bashrc and scripts for students to start, stop, and SSH to RL nodes as if they were running on machines or VMs instead of containers.

```bash
cd ~

cat << EOF > vnc_docker/bashrc
source \$STARTUPDIR/generate_container_user
alias ssh_node="ssh trainee@\$EX_IP"
alias ssh_n1="ssh -t trainee@\$EX_IP docker exec -it n1 bash "
alias ssh_n2="ssh -t trainee@\$EX_IP docker exec -it n2 bash "
alias ssh_n3="ssh -t trainee@\$EX_IP docker exec -it n3 bash "
alias ssh_s1="ssh -t trainee@\$EX_IP docker exec -it s1 bash "
alias ssh_s2="ssh -t trainee@\$EX_IP docker exec -it s2 bash "
alias ssh_s3="ssh -t trainee@\$EX_IP docker exec -it s3 bash "

alias start_n1="ssh -t trainee@\$EX_IP docker start n1 "
alias start_n2="ssh -t trainee@\$EX_IP docker start n2 "
alias start_n3="ssh -t trainee@\$EX_IP docker start n3 "

alias stop_n1="ssh -t trainee@\$EX_IP docker stop n1 "
alias stop_n2="ssh -t trainee@\$EX_IP docker stop n2 "
alias stop_n3="ssh -t trainee@\$EX_IP docker stop n3 "

alias start_s1="ssh -t trainee@\$EX_IP docker start s1 "
alias start_s2="ssh -t trainee@\$EX_IP docker start s2 "
alias start_s3="ssh -t trainee@\$EX_IP docker start s3 "

alias stop_s1="ssh -t trainee@\$EX_IP docker stop s1 "
alias stop_s2="ssh -t trainee@\$EX_IP docker stop s2 "
alias stop_s3="ssh -t trainee@\$EX_IP docker stop s3 "

alias restart_north_nodes="ssh -t trainee@\$EX_IP ./scripts/restart_north_nodes.sh "
alias restart_south_nodes="ssh -t trainee@\$EX_IP ./scripts/restart_south_nodes.sh "
alias create_north_cluster="ssh -t trainee@\$EX_IP ./scripts/create_north_cluster.sh "
alias create_south_cluster="ssh -t trainee@\$EX_IP ./scripts/create_south_cluster.sh "
EOF

cat << EOF > scripts/restart_north_nodes.sh
docker kill n1; docker rm n1;
docker kill n2; docker rm n2;
docker kill n3; docker rm n3;
docker run -d --cap-add=ALL --name n1 -v ./resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n1.redislabs.org --net redislabs --ip 172.18.0.21 redislabs/redis
docker run -d --cap-add=ALL --name n2 -v ./resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n2.redislabs.org --net redislabs --ip 172.18.0.22 redislabs/redis
docker run -d --cap-add=ALL --name n3 -v ./resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n3.redislabs.org --net redislabs --ip 172.18.0.23 redislabs/redis
docker exec --user root n1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
EOF

cat << EOF > scripts/restart_south_nodes.sh
docker kill s1; docker rm s1;
docker kill s2; docker rm s2;
docker kill s3; docker rm s3;
docker run -d --cap-add=ALL --name s1 -v ./resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s1.redislabs.org --net redislabs --ip 172.18.0.31 redislabs/redis
docker run -d --cap-add=ALL --name s2 -v ./resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s2.redislabs.org --net redislabs --ip 172.18.0.32 redislabs/redis
docker run -d --cap-add=ALL --name s3 -v ./resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s3.redislabs.org --net redislabs --ip 172.18.0.33 redislabs/redis
docker exec --user root s1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
EOF

cat << EOF > scripts/create_north_cluster.sh
docker exec -it n1 bash -c "/opt/redislabs/bin/rladmin cluster create persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.21 \
        name north.redislabs.org username admin@redislabs.org password admin";

docker exec -it n2 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.22 \
        username admin@redislabs.org password admin nodes 172.18.0.21";

docker exec -it n3 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.23 \
        username admin@redislabs.org password admin nodes 172.18.0.21";
EOF

cat << EOF > scripts/create_south_cluster.sh
sudo docker exec -it s1 bash -c "/opt/redislabs/bin/rladmin cluster create persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.31 \
        name south.redislabs.org username admin@redislabs.org password admin";
sudo docker exec -it s2 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.32 \
        username admin@redislabs.org password admin nodes 172.18.0.31";
sudo docker exec -it s3 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.33 \
        username admin@redislabs.org password admin nodes 172.18.0.31";
EOF

# make the scripts executable
chmod 755 scripts/restart_north_nodes.sh
chmod 755 scripts/restart_south_nodes.sh
chmod 755 scripts/create_north_cluster.sh
chmod 755 scripts/create_south_cluster.sh
```

12. Add color prompts to RL nodes so students know where they are (cyan@green:/blue$).
```
# colors are: green=32, yellow=33, blue=34, pink=35, cyan=36
# look for '36m..\u', '32m..\h', ':\..34m' 
if [ "$color_prompt" = yes ]; then
   PS1='${debian_chroot:+($debian_chroot)}\[\033[01;36m\]\u\[\033[00m\]@\[\033[01;32m\]\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
   #PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
```

You're finished creating the base VM.

13. Create a snapshot from the base VM called 'reat-snap'.

14. Create an image from the snapshot called 'reat-image'.

15. Create a test instance from the image. The startup script runs the VNC container and passes in the VM instance's private IP so students can SSH from the VNC container to the base image.

Startup script:
```bash
docker run -e EX_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'` -p 80:6901 -e VNC_PW=trainee! --net redislabs --ip 172.18.0.2  --name vnc -d re-vnc;
```

Requirement  | Specification  
------------ | -------------
Name | reat-instance
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | Custom Images > reat-image
Size | 30 GB
Networking | reat-vpc
Startup script | see above

Test the instance.

16. Point a laptop browser to the VM's public IP.

17. Sign in to VNC desktop with password 'trainee!'.

18. In VNC desktop, open the Chrome browser and point it to the RL admin consoles:
North cluster:
- n1 = 172.18.0.21
- n2 = 172.18.0.22
- n3 = 172.18.0.23

South cluster:
- s1 = 172.18.0.31
- s2 = 172.18.0.32
- s3 = 172.18.0.33

19. Open a command-line terminal from the 'Applications' drop-down (top-left) and run the following alias commands to restart nodes and create clusters:

```bash
restart_north_nodes
restart_south_nodes
create_north_cluster
create_south_cluster
```

20. Run the following alias commands to SSH to various destinations:

```bash
ssh_node (this gets you to the main VM instance as the 'trainee' user for the installation lab)
ssh_n1 (these get you to the RL nodes where you can run 'rladmin' commands to view clsuter state)
ssh_n2
ssh_n3
```
