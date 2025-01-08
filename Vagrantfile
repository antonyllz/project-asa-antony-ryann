Vagrant.configure("2") do |config|
  config.vm.box = "generic/debian12"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "p01_Antony_p02_Ryann"

    # RAM
    vb.memory = 1024
  end

    #  Discos adicionais
  config.vm.disk :disk, size: "10GB", name: "disk1"
  config.vm.disk :disk, size: "10GB", name: "disk2"
  config.vm.disk :disk, size: "10GB", name: "disk3"

  config.vm.hostname = "p01_Antony_p02_Ryann"

  config.vm.network "private_network", ip: "192.168.57.10"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
