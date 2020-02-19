# RL Admin Training VM Setup

1. Create a VPC in GPC with a custom subnet (172.18.0.0/16) in the region you want your VMs to run:

Requirement | Specification
------------|--------------
Name | reat-vpc
Subnet Creation Mode | Custom
Subnet Name | reat-subnet
Subnet Region | us-west1
Subnet IP Address Range | 172.18.0.0/16

2. Create a firewall rule for the VPC to allow ingress on all ports from all source instances (0.0.0.0/0) to all targets.
 
3. Create a VM instance on GCP with following
  
Requirement  | Specification  
------------ | -------------
Name | reat-ubuntu-ip1
Region | us-west1
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB
Networking | reat-vpc
  
4. SSH to your VM using GCP  console.

5. Install wetty and set up user "trainee" with password "P@ssword"
```bash 

sudo su
apt -y update

#install wetty - Terminal over HTTP and HTTPS
apt -y install npm
npm install -g wetty

#install and configure supervisor to always run wetty
apt -y install supervisor

#install vi 
apt -y install vim

#create wetty.conf with vi as follows
cd /etc/supervisor/conf.d
vi wetty.conf

#add the following lines in wetty.conf (port 21000 in VM for IP-1, port 13000 in VM for IP-2)
[program:wetty]
command=wetty -p 21000 

#reload wetty with the new config
supervisorctl reload wetty

#save your changes to wetty.conf

#add the trainee user to docker group so it can start and stop docker containers that include Redis Labs software
adduser --disabled-password --gecos "" trainee
echo -e "PASSWORD\nPASSWORD" | sudo passwd trainee 
groupadd docker
usermod -aG docker $USER
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
systemctl restart sshd

#test wetty working with the trainee user. Point your browser to (:21000 for VM IP-1, :13000 for VM IP-2):
http://<vm-ip>:21000/wetty

```
6. install docker-ce 

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

7. Set the Docker training env

```bash

# create a docker subnet
mkdir /opt/redislabs
echo 'nameserver 172.18.0.20' >  /opt/redislabs/resolv.conf
docker network create --subnet=172.18.0.0/16 redislabs

# create the DNS 
docker run --name bind -d -v /opt/redislabs/resolv.conf:/etc/resolv.conf  --net redislabs --restart=always -p 10000:10000/tcp   --ip 172.18.0.20 rahimre/redislabs-training-bind

# create the north cluster

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

8. Set the default editor to vim

```
 update-alternatives --config editor
 ```
 
9. add the following line to /etc/sudoers using "sudo visudo" so 'trainee' user can run sudo

```
trainee ALL=(ALL) NOPASSWD:ALL

```

10. add the following to user trainee's .bashrc 

```
#sign in to wetty as trainee
#add the following aliases to trainee's .bashrc file

alias n1="sudo docker exec -it n1 bash "
alias n2="sudo docker exec -it n2 bash "
alias n3="sudo docker exec -it n3 bash "
alias s1="sudo docker exec -it s1 bash "
alias s2="sudo docker exec -it s2 bash "
alias s3="sudo docker exec -it s3 bash "
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

   



