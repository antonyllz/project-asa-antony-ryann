Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vbguest.auto_update = false
  

  config.vm.provider "virtualbox" do |vb|
    vb.name = "p01-Antony-p02-Ryann"

    # RAM
    vb.memory = 1024
  end

    #  Discos adicionais
   
    # Adicionar 3 discos adicionais de 10GB
   (0..2).each do |i|
    config.vm.disk :disk, size: "10GB", name: "disk-#{i}"
   end

  config.vm.hostname = "p01-Antony-p02-Ryann"

  config.vm.network "private_network", ip: "192.168.57.10"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.compatibility_mode = "2.0"
  end
end
