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
  config.vm.define "rasvpn-server" do |rasvpn-server|
    rasvpn-server.vm.hostname = "rasvpn-server.ras"
    rasvpn-server.vm.network "private_network", ip: "192.168.20.10"
  end
  config.vm.define "rasvpn-client" do |rasvpn-client|
    rasvpn-client.vm.hostname = "rasvpn.loc"
    rasvpn-client.vm.network "private_network", ip: "192.168.20.20"
    rasvpn-client.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/playbook.yml"
      ansible.inventory_path = "ansible/hosts"
      ansible.host_key_checking = "false"
      ansible.limit = "all"
    end
  end
end

