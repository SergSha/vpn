# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.define "server" do |server|
    server.vm.hostname = "server.loc"
    server.vm.network "private_network", ip: "192.168.10.10"
  end
  config.vm.define "client" do |client|
    client.vm.hostname = "client.loc"
    client.vm.network "private_network", ip: "192.168.10.20"
  end
  config.vm.define "rasvpn" do |rasvpn|
    rasvpn.vm.hostname = "rasvpn.ras"
    rasvpn.vm.network "private_network", ip: "192.168.20.10"
  end
  config.vm.define "rasclient" do |rasclient|
    rasclient.vm.hostname = "rasclient.ras"
    rasclient.vm.network "private_network", ip: "192.168.20.20"
    rasclient.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/playbook.yml"
      ansible.inventory_path = "ansible/hosts"
      ansible.host_key_checking = "false"
      ansible.limit = "all"
    end
  end
end

