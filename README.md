
# k8s-pi-project
NOT READY 



Install kubernetes on a 6 raspberry Pi cluster with flannel network

## bootstrap on each Raspberry PI
```sh
ssh-copy-id pi@raspberrypi.local
ssh pi@raspberrypi.local
sudo apt update
sudo apt -qy install vim curl apt-transport-https ca-certificates curl git
```
### raspi-config
```sh
sudo raspi-config
```
This section could probably be automated

* change user password
* Localisation Options:
  * Change local --> en_US. UTF-8 UTF-8
  * Change time zone --> Montreal
  * Change Wifi country --> CA
* Advanced Options
  * Memory SPlit --> 16

```sh
echo 'LANG=en_US.UTF-8
LC_ALL=en_US.UTF-8
LANGUAGE=en_US.UTF-8
' | sudo tee /etc/default/locale

sudo apt -y upgrade
sudo reboot
```

## On each node
```sh 
ssh pi@raspberrypi.local

# cluster variables
export dns="192.168.7.1"
export master1_ip="192.168.7.220"
export master2_ip="192.168.7.221"
export worker1_ip="192.168.7.222"
export worker2_ip="192.168.7.223"
export worker3_ip="192.168.7.224"
export worker4_ip="192.168.7.225"
```

## master1
```sh
machine="master1"
ip=${master1_ip}

# set hostname
sudo hostnamectl --transient set-hostname ${machine}
sudo hostnamectl --static set-hostname ${machine}
sudo hostnamectl --pretty set-hostname ${machine}
sudo sed -i s=raspberrypi=${machine}=g /etc/hosts

# set hosts file
echo "
# K8S nodes
${master1_ip} master1

${worker1_ip} worker1
${worker2_ip} worker2
${worker3_ip} worker3
" | sudo tee -a /etc/hosts

# set network
sudo sed -i s=^hostname=${machine}=g /etc/dhcpcd.conf
echo "interface eth0
static ip_address=$ip/24
static routers=$dns
static domain_name_servers=$dns" | sudo tee -a /etc/dhcpcd.conf

# Install Docker 17.x
curl -fsSL get.docker.com -o get-docker.sh
export DOWNLOAD_URL="https://download.docker.com"
export CHANNEL="stable"
export VERSION="17.12.1"
chmod +x get-docker.sh
./get-docker.sh
sudo usermod pi -aG docker
newgrp - docker

# lock docker version
sudo apt-mark hold docker-ce

# Disable Swap
sudo dphys-swapfile swapoff && \
sudo dphys-swapfile uninstall && \
sudo update-rc.d dphys-swapfile remove

# Fix cgroup
sudo cp /boot/cmdline.txt /boot/cmdline_backup.txt
orig="$(head -n1 /boot/cmdline.txt) cgroup_enable=memory cgroup_memory=1"
echo $orig | sudo tee /boot/cmdline.txt

# Add repo list and install kubeadm
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
  echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
  sudo apt-get update -q && \
  sudo apt-get install -qy kubeadm

# lock kubeadm version
sudo apt-mark hold kubeadm

# install and configure golang 1.9
wget https://storage.googleapis.com/golang/go1.9.linux-armv6l.tar.gz
sudo tar -C /usr/local -xzf go1.9.linux-armv6l.tar.gz
echo '
export GOPATH=$HOME/go
export PATH=/usr/local/go/bin:$PATH:$GOPATH/bin
'| tee -a  $HOME/.bashrc  $HOME/.profile
go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
sudo ln -s /home/pi/go/bin/crictl  /usr/local/bin/

# active kernel settings
sudo reboot
```

## worker1


 




## On master1

### cluster init
```sh
ssh pi@master1

# deploy master
echo "apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
controllerManagerExtraArgs:
  pod-eviction-timeout: 10s
  node-monitor-grace-period: 10s" | tee $HOME/kubeadm.yaml

sudo kubeadm init --config kubeadm.yaml

# install weaver net
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

kubectl get nodes # check if master ready

# last step
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```sh
# install network layer

```

## On each workers