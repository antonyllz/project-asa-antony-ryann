Vagrant.configure("2") do |config|
  config.vm.box = "generic/debian12"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "p01-Antony-p02-Ryann"

    # RAM
    vb.memory = 1024
  end

    #  Discos adicionais
  config.vm.disk :disk, size: "10GB", name: "sdb"
  config.vm.disk :disk, size: "10GB", name: "sdc"
  config.vm.disk :disk, size: "10GB", name: "sdd"

  config.vm.hostname = "p01-Antony-p02-Ryann"

  config.vm.network "private_network", ip: "192.168.57.10"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
