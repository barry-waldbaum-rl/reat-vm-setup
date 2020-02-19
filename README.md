# REAT VM Setup

1. Create a GCP VPC with custom subnet 172.18.0.0/16 in the region where you want your VM instances to run.

Requirement | Specification
------------|--------------
Name | reat-vpc
Subnet Creation Mode | Custom
Subnet Name | reat-subnet
Subnet IP Address Range | 172.18.0.0/16

2. Create a firewall rule for your VPC that allows ingress on all ports from all sources (0.0.0.0/0) to all targets.
 
3. Create a base VM image in the region where you want your instances to run (at the end of this setup page, you will take a GCP snapshot of this base VM, generate a VM image from the snapshot, and save that image to create VM instances for students in the future).
  
Requirement  | Specification  
------------ | -------------
Name | reat-vm
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB
Networking | reat-vpc
  
4. SSH to your base image VM using GCP console.

5. Install 'vi' and set up the 'trainee' user with password 'P@ssword' and the ability to run 'sudo'.

```bash 
sudo su
apt -y update
apt -y install vim

# Set the default editor to vim
update-alternatives --config editor

#add "trainee" user to Docker group so it can start and stop containers with Redis software
adduser --disabled-password --gecos "" trainee
echo -e "PASSWORD\nPASSWORD" | sudo passwd trainee 
groupadd docker
usermod -aG docker $USER
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config

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

7. Switch to the "trainee" user and its home directory to finish setup.

```bash
sudo su - trainee
```

7. Create a Docker subnet and add a DNS bind server to it.

```bash
mkdir resolve
echo 'nameserver 172.18.0.20' > ./resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 redislabs

docker run --name bind -d -v ./resolve/resolv.conf:/etc/resolv.conf  --net redislabs --restart=always -p 10000:10000/tcp   --ip 172.18.0.20 rahimre/redislabs-training-bind
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
cat << EOF > vnc_docker/Dockerfile
## Custom Dockerfile
FROM consol/ubuntu-xfce-vnc

# Switch to root user to install additional software
USER 0

## Install a gedit
RUN apt update; apt install -y ssh;
RUN mkdir /headless/.ssh
COPY ./ssh /headless/.ssh
RUN chown -R 1000 /headless/.ssh/
COPY bashrc /headless/.bashrc
RUN chown -R 1000 /headless/.bashrc

## switch back to default user
USER 1000
EOF
```

10. Create a bashrc file that will run when a user signs into VNC. Aliases allow the user to start, stop, and ssh to nodes as if they were running on machines or VMs instead of containers.

```bash
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

alias reset_north_nodes="ssh -t trainee@\$EX_IP ./scripts/reset_north_nodes.sh "
alias reset_south_nodes="ssh -t trainee@\$EX_IP ./scripts/reset_south_nodes.sh "
alias create_north_cluster="ssh -t trainee@\$EX_IP ./scripts/create_north_cluster.sh "
alias create_south_cluster="ssh -t trainee@\$EX_IP ./scripts/create_south_cluster.sh "
EOF
```

11. Build the Docker image from the Dockerfile that adds VNC software to the VM.

```bash
cd vnc_docker
docker build -t re-vnc .
```

12. Run a Docker container from the Docker image that adds VNC software to the VM. VNC will allow the student to sign in to their VM instance's desktop on port 80 using the VM's public IP address (which can be found in GCP console after the student's VM instance is started).

```bash
#
# You can uncomment and run the following commands before running the next command if you need to reset the base VM image
# and create an updated snapshot
#
# docker stop vnc; docker rm vnc;
#
docker run -e EX_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'` -p 80:6901 -e VNC_PW=trainee! --net redislabs --ip 172.18.0.2  --name vnc -d re-vnc;
```

docker run -d  --cap-add=ALL --name n1  -v /opt/redislabs/resolv.conf:/etc/resolv.conf  -p 21443:8443 -p 41443:9443 --restart=always  --hostname  n1.north.redislabs-training.org --net redislabs --ip 172.18.0.21  redislabs/redis
docker run -d  --cap-add=ALL --name n2  -v /opt/redislabs/resolv.conf:/etc/resolv.conf  -p 22443:8443 -p 42443:9443 --restart=always  --hostname  n2.north.redislabs-training.org  --net redislabs --ip 172.18.0.22   redislabs/redis
docker run -d  --cap-add=ALL --name n3  -v /opt/redislabs/resolv.conf:/etc/resolv.conf -p 23443:8443 -p 43443:9443 --restart=always  --hostname  n3.north.redislabs-training.org  --net redislabs --ip 172.18.0.23    redislabs/redis

docker exec --user root n1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"

# create the south cluster

docker run -d  --cap-add=ALL  -v /opt/redislabs/resolv.conf:/etc/resolv.conf --name s1 -p 31443:8443 -p 51443:9443 --restart=always --hostname  s1.south.redislabs-training.org   --net redislabs --ip 172.18.0.31  redislabs/redis
docker run -d  --cap-add=ALL  -v /opt/redislabs/resolv.conf:/etc/resolv.conf --name s2 -p 32443:8443 -p 52443:9443 --restart=always --hostname s2.south.redislabs-training.org   --net redislabs --ip 172.18.0.32  redislabs/redis
docker run -d  --cap-add=ALL  -v /opt/redislabs/resolv.conf:/etc/resolv.conf --name s3 -p 33443:8443 -p 53443:9443 --restart=always  --hostname s3.south.redislabs-training.org   --net redislabs --ip 172.18.0.33   redislabs/redis

docker exec --user root s1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
   
```

10. add the following to user trainee's .bashrc 

```

#add the following aliases to trainee's .bashrc file
alias ns="sudo docker run -it --net redislabs   -v /opt/redislabs/resolv.conf:/etc/resolv.conf   tutum/dnsutils bash"
alias crl="sudo docker run -it --net redislabs   -v /opt/redislabs/resolv.conf:/etc/resolv.conf   tutum/curl bash"
alias redis="sudo docker run -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /home/trainee/redis:/data --net redislabs  -it redis bash "
```

11. update trainee's .bashrc file to use the following lines so VM shell prompt color (cyan@green:/blue$) stands out from container shell color and students can keep track of which system they're signed into (they shift between VM and RES containers):
```
# colors are: green=32, yellow=33, blue=34, pink=35, cyan=36
# look for '36m..\u', '32m..\h', ':\..34m' 
if [ "$color_prompt" = yes ]; then
   PS1='${debian_chroot:+($debian_chroot)}\[\033[01;36m\]\u\[\033[00m\]@\[\033[01;32m\]\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
   #PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
```

12. add the following scripts to trainee's root directory (get exact code from Jim, 'create' code is similar to code above, prefixing each cmd with 'sudo', adding 'docker kill' cmds):
* reset_north_cluster.sh
* reset_south_cluster.sh
* create_north_cluster.sh
* create_south_cluster.sh

13. make the scripts executable:
```
chmod 755 reset_north_cluster.sh
chmod 755 reset_south_cluster.sh
chmod 755 create_north_cluster.sh
chmod 755 create_south_cluster.sh
```

14. Take a snapshot of the running instance and create training image called "reat-snap-ip1" from the snapshot

15. Go to Images on the GCP menu, find image "reat-image-ip1" and create a new training instance 'reat-instance-ip1'.

16. You can log into the instance using ssh client 
```
  ssh trainee@publicip
```  
   
   Now you can access the instance from browser 
   http://publicip:21000/wetty
   
17. Go to the bottom of VM creation page and click ‘command line’ to get the gcloud command to create an image that’s scriptable for a class of many.

   



