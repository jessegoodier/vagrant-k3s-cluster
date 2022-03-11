# -*- mode: ruby -*-
# vi: set ft=ruby :
# encoding: UTF-8

number_of_agent_nodes=1
k3s_version="v1.22.7+k3s1" #replace all. 
k3s_install_script="https://raw.githubusercontent.com/k3s-io/k3s/$k3s_version/install.sh"


# Extra parameters in INSTALL_K3S_EXEC variable because of
# K3s was picking up the nat interface when starting server and node
# https://github.com/alexellis/k3sup/issues/306

server_script = <<-SHELL
    sudo -i
    ! [ -x "$(command -v curl)" ] && apt update && apt install -y curl
    export INSTALL_K3S_EXEC="--flannel-iface=enp0s8" # make sure this matches the adapter your vm creates
    curl -sfL $k3s_install_script | sh -
    echo "Sleeping for 5 seconds to wait for k3s to start"
    sleep 5
    cp /var/lib/rancher/k3s/server/token /vagrant_shared
    sed -i 's/127.0.0.1/k3server/' /etc/rancher/k3s/k3s.yaml
    cp /etc/rancher/k3s/k3s.yaml /vagrant_shared
    
SHELL

node_script = <<-SHELL
cat <<-'EOF' > /home/vagrant/agent.sh
! [ -x "$(command -v curl)" ] && sudo apt update && sudo apt install -y curl
export K3S_TOKEN_FILE=/home/vagrant/token
export K3S_URL=https://$(getent hosts k3server | cut -d " " -f1):6443
export INSTALL_K3S_EXEC="--flannel-iface=enp0s8" #make sure this matches the adapter your vm creates
echo \$k3s_version
echo $k3s_install_script
AGENT_INSTALLED="false"
curl -sfL $k3s_install_script > install.sh
while [ $AGENT_INSTALLED != "true" ]
  do
    if curl -k 'https://k3server:6443/' | grep -q 'Unauthorized' ; then
      AGENT_INSTALLED="true"
      sh install.sh
      echo "Installed"
    else
      echo "server not responding yet"
      sleep 5
    fi
  done
EOF
      chmod a+x /home/vagrant/agent.sh
      cp /vagrant_shared/token /home/vagrant/token
      echo "@reboot sh /home/vagrant/agent.sh > /home/vagrant/agent-cron.log 2>&1" | crontab -
SHELL

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"

  config.vm.define "k3server", primary: true do |server|
    server.vm.network "public_network", bridge: "Intel(R) Ethernet Connection (7) I219-V", use_dhcp_assigned_default_route: true
    server.vm.synced_folder "./shared", "/vagrant_shared"
    server.vm.hostname = "k3server"
    server.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = "2"
    end
    server.vm.provision "shell", inline: server_script
  end

  (1..number_of_agent_nodes).each do |node_num|
    config.vm.define "node#{node_num}" do |node|
      node.vm.network "public_network", bridge: "Intel(R) Ethernet Connection (7) I219-V", use_dhcp_assigned_default_route: true
      node.vm.synced_folder "./shared", "/vagrant_shared"
      node.vm.hostname = "node#{node_num}"
      node.vm.provider "virtualbox" do |vb|
        vb.memory = "1024"
        vb.cpus = "1"
      end
      node.vm.provision "shell", inline: node_script
    end
  end
end