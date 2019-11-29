# -*- mode: ruby -*-
# vi: set ft=ruby :

# :box_version => "20180831.0.0",
# https://app.vagrantup.com/ubuntu/boxes/xenial64/versions/20191005.0.0
#        :box => "ubuntu/xenial64",
#        :box_version => "20191005.0.0",

#  config.vm.box = "ubuntu/bionic64"
#  config.vm.box_version = "20191125.0.0"
servers = [
    {
        :name => "k8s-head",
        :type => "master",
        :box => "ubuntu/bionic64",
        :box_version => "20191125.0.0",
        :eth1 => "192.168.33.11",
        :mem => "2048",
        :cpu => "2"
    },
   {
        :name => "k8s-node-1",
        :type => "node",
        :box => "ubuntu/bionic64",
        :box_version => "20191125.0.0",
        :eth1 => "192.168.33.12",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-2",
        :type => "node",
        :box => "ubuntu/bionic64",
        :box_version => "20191125.0.0",
        :eth1 => "192.168.33.13",
        :mem => "2048",
        :cpu => "2"
    }
]

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT
    echo  "Starting to execute configureBox"
    # install docker v18.09
    # reason for not using docker provision is that it always installs latest version of the docker, but kubeadm requires 18.09 or older
    apt-get update
    # Install Docker CE
    ## Set up the repository:
    ### Install packages to allow apt to use a repository over HTTPS
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    ### Add Docker’s official GPG key
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

    echo  "doing the docker now"
    ### Add Docker apt repository.
    add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
    ## Install Docker CE.
    echo  "install docker"
    apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 18.09 | head -1 | awk '{print $3}')

    echo  "setup docker daemon"
    # Setup daemon.
    cat > /etc/docker/daemon.json <<EOF
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
      "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
EOF

    echo "making docker systemd directory"
    mkdir -p /etc/systemd/system/docker.service.d
    ls -al /etc/systemd/system/

    # Restart docker.
    systemctl daemon-reload
    systemctl restart docker

    echo "creating rc.local to add routing and iptables stuff"
    route add -net 192.168.90.0/24 gw 192.168.33.1

    echo "adding the host network route soe we can access the load balancer"
    route add -net 192.168.102.0/24 gw 192.168.33.1

    cat <<EOF > /etc/rc.local
    #!/bin/sh -e
    #
    route add -net 192.168.90.0/24 gw 192.168.33.1
    # host network entry allows us to use router box to forward from linux host
    # also need to add a route on the linux host sudo route add -net 192.168.90.0/24 gw 192.168.102.254
    route add -net 192.168.102.0/24 gw 192.168.33.1

    exit 0
EOF
      chmod 0755 /etc/rc.local
      # for kube-proxy (iptables)
      #
      echo net.bridge.bridge-nf-call-iptables=1 >> /etc/sysctl.conf
      modprobe br_netfilter
      sysctl -p

    # run docker commands as vagrant user (sudo not required)
    echo  "adding vagrant to the docker group"
    usermod -aG docker vagrant

    # install kubeadm
    echo "getting prerequisites for kubeadm"
    apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -


    echo "installing the kubernetes repo"
    apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
    echo "starting the kube stuff"
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl
    echo  "finished doing the kube stuff"

    # kubelet requires swap off
    echo "kubelet requires swap off"
    swapoff -a

    # keep swap off after reboot
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

    # ip of this box
    #IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
    IP_ADDR=$(ip a show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f1)
    # set node-ip
    echo "setting default values for kubelet"
    ls -al /etc/default
    touch /etc/default/kubelet
    chmod 644 /etc/default/kubelet
    # this command does not work on emtpy file
    # sudo sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" /etc/default/kubelet
    # use this one instead. This fixes the proper ip address for worker nodes and fixes
    # issues with accessing logs
    # reference https://medium.com/@joatmon08/playing-with-kubeadm-in-vagrant-machines-part-2-bac431095706
    echo "KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" | sudo tee /etc/default/kubelet
    sudo systemctl restart kubelet
    sudo apt install inetutils-traceroute
    echo  "finished configureBox"
SCRIPT

$configureMaster = <<-SCRIPT
    echo "This is master"
    # ip of this box
    #IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
    IP_ADDR=$(ip a show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f1)

    # install k8s master
    HOST_NAME=$(hostname -s)
    echo "Executing kubeadm init"
    kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=10.244.0.0/16

    #copying credentials to regular user - vagrant
    echo "copying credentials to regular user - vagrant"
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    echo "Executing ls -al /etc/kubernetes"
    ls -al /etc/kubernetes
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

    export KUBECONFIG=/etc/kubernetes/admin.conf

    echo "Executing Flannel pod network addon"
    #kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ## Deploy flannel, custom file with updated iface name and k8s network cidr
    kubectl apply -f /vagrant/flannel/kube-flannel.yml

    # apply the metallb load balancer
  echo "Executing METALLB  addon"
  curl -s https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml >  /root/metallb.yaml
  sed  -i '1i ---' /root/metallb.yaml
  cat << EOF > /root/metallb_configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - my-asn: 64522
      peer-asn: 64512
      peer-address: 192.168.33.1
      peer-port: 179
      router-id: 192.168.33.1
    address-pools:
    - name: my-ip-space
      protocol: bgp
      avoid-buggy-ips: true
      addresses:
      - 192.168.90.192/26
EOF
  kubectl apply -f /root/metallb.yaml
  kubectl apply -f /root/metallb_configmap.yaml

  # deployment of an example LoadBalancer type
  echo "Creating an example LoadBalancer service to demonstrate the load balancer of metallb"
  kubectl run source-ip-app --image=k8s.gcr.io/echoserver:1.4
  kubectl expose deployment source-ip-app --name=loadbalancer --port=80 --target-port=8080 --type=LoadBalancer
  kubectl get svc loadbalancer
  kubectl patch svc loadbalancer -p '{"spec":{"externalTrafficPolicy":"Local"}}'

    # create the join cluster command for the slave nodes to execute
    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh

    # required for setting up password less ssh between guest VMs
    echo "Executing ssh changes for vagrant user"
    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart
    echo  "finished configureMaster"

SCRIPT

$configureNode = <<-SCRIPT
    echo "This is worker"
    apt-get install -y sshpass
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.33.11:/etc/kubeadm_join_cmd.sh .
    echo  "configureNode joing cluster"
    sh ./kubeadm_join_cmd.sh
    echo  "finished configureNode"
SCRIPT

$configureRouter=<<-SCRIPT
  apt-get update
  apt-get install -y curl bird
  mv /etc/bird/bird.conf /etc/bird/bird.conf.original
  cat <<EOF > /etc/bird/bird.conf
router id 192.168.33.1;
protocol direct {
  interface "lo"; # Restrict network interfaces BIRD works with
}
protocol kernel {
  persist; # Don't remove routes on bird shutdown
  scan time 20; # Scan kernel routing table every 20 seconds
  import all; # Default is import all
  export all; # Default is export none
}
# This pseudo-protocol watches all interface up/down events.
protocol device {
  scan time 10; # Scan interfaces every 10 seconds
}
protocol bgp peer2 {
  local as 64512;
  neighbor 192.168.33.11 as 64522;
  import all;
  export all;
}
protocol bgp peer1 {
  local as 64512;
  neighbor 192.168.33.12 as 64522;
  import all;
  export all;
}
protocol bgp peer3 {
  local as 64512;
  neighbor 192.168.33.13 as 64522;
  import all;
  export all;
}
EOF
  service bird restart
  echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/55-kubeadm.conf
  sysctl -p /etc/sysctl.d/55-kubeadm.conf
  sudo apt install inetutils-traceroute
SCRIPT

Vagrant.configure("2") do |config|

    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1] , virtualbox__intnet: "k8s-metallb-net"

            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
            	v.customize ["modifyvm", :id, "--groups", "/K8S MetalLB Dev"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            # we cannot use this because we can't install the docker version we want - https://github.com/hashicorp/vagrant/issues/4871
            #config.vm.provision "docker"

            config.vm.provision "shell", inline: $configureBox

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureNode
            end
        end
    end
 # now configure a router
  config.vm.define "router" do |router|
    router.vm.box = "ubuntu/bionic64"
    router.vm.network "public_network", bridge: "em1", ip: "192.168.102.254"
    router.vm.network "private_network",
                       ip: "192.168.33.1",
                       netmask: "255.255.255.0",
                       auto_config: true,
                       virtualbox__intnet: "k8s-metallb-net"
    router.vm.network "private_network",
                       ip: "192.168.90.1",
                       netmask: "255.255.255.0",
                       auto_config: true,
                       virtualbox__intnet: "client-metallb-net"
     router.vm.host_name = "router"
     router.ssh.insert_key = false
     router.vm.provision "shell", inline: $configureRouter
     router.vm.provider "virtualbox" do |v|
       v.name = "router"
       v.customize ["modifyvm", :id, "--ostype", "Debian_64"]
       v.customize ["modifyvm", :id, "--groups", "/K8S MetalLB Dev"]
       v.cpus = 1
       v.memory = 512
     end
  end

end 
