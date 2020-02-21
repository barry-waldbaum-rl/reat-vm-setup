# REAT VM Setup

The goal is to setup one GCP VM that runs a local Docker network with:
- VNC server for students to sign into
- A DNS bind server for managing hostname lookups
- 2 sets of 3 containers (each runs a Redis Enterprise node that students will start and group into 2 RES clusters).

You are going to do the following:
- Build a GCP base VM with all the software and configuration
- Take a snapshot of the base VM and an image of the snapshot
- Create 1 or more VM instances from the image for students to use.

Steps:

1. Create a GCP VPC with custom subnet 172.18.0.0/16 in the region where you want your VM instances to run.

Requirement | Specification
------------|--------------
Name | reat-vpc
Subnet Creation Mode | Custom
Subnet Name | reat-subnet
Subnet IP Address Range | 172.18.0.0/16

2. Create a firewall rule for your VPC that allows ingress on all ports from all sources (0.0.0.0/0) to all targets.
 
3. Create a base VM in the region where you want your instances to run (at the end of this setup page, you will take a GCP snapshot of this VM, generate a VM image from the snapshot, and save that image to create VM instances for students in the future).
  
Requirement  | Specification  
------------ | -------------
Name | reat-base-vm
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB
Networking | reat-vpc
  
4. SSH to your base image VM using GCP console.

5. Switch to 'root' user, install 'vi', and set up the 'trainee' user with password 'P@ssword' and the ability to run 'sudo'.

```bash 
sudo su
apt -y update
apt -y install vim

# Set the default editor to vim
update-alternatives --config editor

# Add 'trainee' user to 'docker' group so it can start and stop containers that run Redis software
adduser --disabled-password --gecos "" trainee
groupadd docker
usermod -aG docker $USER
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

7. Switch to the 'trainee' user and its home directory to finish setup.

```bash
sudo su - trainee
```

7. Create a Docker subnet and add a DNS bind server to it.

```bash
mkdir resolve
echo 'nameserver 172.18.0.20' > ./resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 redislabs

docker run --name bind -d -v ~/resolve/resolv.conf:/etc/resolv.conf  --net redislabs --restart=always -p 10000:10000/tcp --ip 172.18.0.20 rahimre/redislabs-training-bind
```

8. Generate keys for students to VNC to their VM instance's desktop and run a browser locally. The browser allows students to  access admin console for Redis Labs nodes running on the VM's Docker subnet using local IP addresses.

```bash
#
# reset
# rm -rf .ssh vnc_docker
#
ssh-keygen -q -t rsa -N '' -f ~/.ssh/id_rsa 2>/dev/null <<< y >/dev/null
cp -r .ssh/id_rsa.pub .ssh/authorized_keys 

mkdir vnc_docker
cp -r .ssh/ vnc_docker/ssh 
```

9. Create the Dockerfile that will be used to add VNC software to the VM.

```bash
cat << EOF > ~/vnc_docker/Dockerfile
## Custom Dockerfile
FROM consol/ubuntu-xfce-vnc

# Switch to root user to install additional software
USER 0

## Install a gedit
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

10. Create a bashrc file that will run when a user signs into VNC. Aliases allow the user to start, stop, and ssh to nodes as if they were running on machines or VMs instead of containers (STARTUPDIR = /dockerstartup/ ).

```bash
cat << EOF > ~/vnc_docker/bashrc
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

alias reset_north_nodes="ssh -t trainee@\$EX_IP /home/trainee/scripts/reset_north_nodes.sh "
alias reset_south_nodes="ssh -t trainee@\$EX_IP /home/trainee/scripts/reset_south_nodes.sh "
alias create_north_cluster="ssh -t trainee@\$EX_IP /home/trainee/scripts/create_north_cluster.sh "
alias create_south_cluster="ssh -t trainee@\$EX_IP /home/trainee/scripts/create_south_cluster.sh "
EOF
```

11. Build the Docker image from the Dockerfile that adds VNC software to the VM.

```bash
cd ~/vnc_docker
docker build -t re-vnc .
```

12. Run a Docker container from the Docker image that adds VNC software to the VM (VNC allows students to sign in to a VM instance desktop on port 80 using the VM's public IP address--which can be found in GCP console after the student's VM instance is startet--so they can configure cluster nodes and databases using an open port rather than port 8443).

```bash
#
# You can uncomment and run the following commands before running the next command if you need to reset the base VM image
# and create an updated snapshot
#
# docker stop vnc; docker rm vnc;
#
docker run -e EX_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'` -p 80:6901 -e VNC_PW=trainee! --net redislabs --ip 172.18.0.2  --name vnc -d re-vnc;
```

NOTE 1: Setting EX_IP in the base VM creates the problem of passing this container environment variable to new VM instances with different IPs. It will have to be corrected somehow like the comment in NOTE 2 below.

13. Create scripts that a student can run to reset Redis Labs nodes and create clusters on their VM instance.

```bash
cat << EOF > ~/scripts/reset_north_nodes.sh
docker kill n1; docker rm n1;
docker kill n2; docker rm n2;
docker kill n3; docker rm n3;
docker run -d --cap-add=ALL --name n1 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n1.redislabs.org --net redislabs --ip 172.18.0.21 redislabs/redis
docker run -d --cap-add=ALL --name n2 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n2.redislabs.org --net redislabs --ip 172.18.0.22 redislabs/redis
docker run -d --cap-add=ALL --name n3 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n3.redislabs.org --net redislabs --ip 172.18.0.23 redislabs/redis
docker exec --user root n1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
EOF

cat << EOF > ~/scripts/reset_south_nodes.sh
docker kill s1; docker rm s1;
docker kill s2; docker rm s2;
docker kill s3; docker rm s3;
docker run -d --cap-add=ALL --name s1 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s1.redislabs.org --net redislabs --ip 172.18.0.31 redislabs/redis
docker run -d --cap-add=ALL --name s2 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s2.redislabs.org --net redislabs --ip 172.18.0.32 redislabs/redis
docker run -d --cap-add=ALL --name s3 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s3.redislabs.org --net redislabs --ip 172.18.0.33 redislabs/redis
docker exec --user root s1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
EOF

cat << EOF > ~/scripts/create_north_cluster.sh
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

cat << EOF > ~/scripts/create_south_cluster.sh
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
chmod 755 ~/scripts/reset_north_nodes.sh
chmod 755 ~/scripts/reset_south_nodes.sh
chmod 755 ~/scripts/create_north_cluster.sh
chmod 755 ~/scripts/create_south_cluster.sh
```

14. add the following to user trainee's .bashrc 

```bash
# add the following aliases to trainee's .bashrc file
alias ns="sudo docker run -it --net redislabs   -v /opt/redislabs/resolv.conf:/etc/resolv.conf   tutum/dnsutils bash"
alias crl="sudo docker run -it --net redislabs   -v /opt/redislabs/resolv.conf:/etc/resolv.conf   tutum/curl bash"
alias redis="sudo docker run -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /home/trainee/redis:/data --net redislabs  -it redis bash "
```

15. update trainee's .bashrc file to use the following lines so VM shell prompt color (cyan@green:/blue$) stands out from container shell color and students can keep track of which system they're signed into (they shift between VM and RES containers):
```
# colors are: green=32, yellow=33, blue=34, pink=35, cyan=36
# look for '36m..\u', '32m..\h', ':\..34m' 
if [ "$color_prompt" = yes ]; then
   PS1='${debian_chroot:+($debian_chroot)}\[\033[01;36m\]\u\[\033[00m\]@\[\033[01;32m\]\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
   #PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
```

Now you are finished creating your base VM that will be used by students.

16. Take a GCP snapshot of the running base VM called 'reat-snap'.

17. Create a GCP image called 'reat-image' from the snapshot.

Now you are ready to create new instances for students.

18. Go to GCP > VM instances, create a new VM instance from 'reat-image' with the following:

Requirement  | Specification  
------------ | -------------
Name | reat-instance
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | from reat-image with 30 GB
Networking | reat-vpc

19. You can go to the bottom of VM creation page and click ‘command line’ to get the gcloud command to create an image that’s scriptable for a class of many.

NOTE 2: Somewhere we will have to do something equivalent to running the following on the new VM instance to correct for NOTE 1 above:

```bash
> sudo docker exec -it vnc bash -c "export EX_IP=172.18.0.19"
```

HOWEVER: The above doesn't stick in the container, just the bash shell. So you might need to edit the container config file in:
```bash
/var/lib/docker/containers/<container-id>/config.json.
```

Now you are ready to test what a student would do with the instance.

20. You can VNC in to the VM instance like a student would by pointing a browser to the instance's public IP and signing in with the password 'trainee!'.

21. From the VNC desktop, you can continue as a student would by opening a terminal window from the 'Applications' drop-down (top-left). This signs you in as the 'default' user of the main VM.

22. In the terminal window, run the following alias to sign in as the 'trainee' user. From there, you can run lab scripts on the VM that can reset Redis Lab nodes or create Redis Labs clusters in containers (called n1-n3 or s1-s3 on the local Docker network):

```bash
ssh_node
```

23. Open a browser and point it to one of the following Redis Labs nodes using their local IP addresses:
Cluster: north.redislabs.org
n1 = 172.18.0.21
n2 = 172.18.0.22
n3 = 172.18.0.23

Cluster: south.redislabs.org
s1 = 172.18.0.31
s2 = 172.18.0.32
s3 = 172.18.0.33
