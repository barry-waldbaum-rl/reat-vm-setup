# REAT VM Setup

1. Create a GCP VPC with custom subnet 172.18.0.0/16 in the region where you want your VM instances to run.

Requirement | Specification
------------|--------------
Name | <vpc-name>
Subnet Creation Mode | Custom
Subnet Name | <subnet-name>
Subnet IP Address Range | 172.18.0.0/16

2. Create a firewall rule for your VPC that allows ingress on all ports from all sources (0.0.0.0/0) to all targets.
 
3. Create a VM in the region where you want your instances to run.
  
Requirement  | Specification  
------------ | -------------
Name | reat-vm
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB
Networking | <vpc-name>
  
4. SSH to your VM using GCP console.

5. Install vi and set up the "trainee" user with password "P@ssword".

```bash 

sudo su
apt -y update
apt -y install vim

#add "trainee" user to Docker group so it can start and stop containers with Redis software
adduser --disabled-password --gecos "" trainee
echo -e "PASSWORD\nPASSWORD" | sudo passwd trainee 
groupadd docker
usermod -aG docker $USER
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config

```

6. Install docker-ce 

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

7. Create a Docker subnet and add a DNS bind server to it

```bash

mkdir resolve
echo 'nameserver 172.18.0.20' > ./resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 redislabs

docker run --name bind -d -v ./resolve/resolv.conf:/etc/resolv.conf  --net redislabs --restart=always -p 10000:10000/tcp   --ip 172.18.0.20 rahimre/redislabs-training-bind

```

8. Generate your VNC sign-in keys for the "trainee" user.

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

9. Make a Dockerfile that will add VNC software to the VM.

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

10. 

north cluster

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

   



