Vagrant.configure("2") do |config|
  config.vm.box = "centos7"
  config.vm.provider "virtualbox" do |v|
	v.memory = 256
	v.cpus = 1
  end
  
  config.vm.define "pgbackup" do |pgbackup|
    pgbackup.vm.network "private_network", ip: "192.168.50.12",
    virtualbox__intnet: "net1"
    pgbackup.vm.hostname = "pgbackup"
	  config.vm.provision "ansible" do |ansible|
        ansible.playbook = "play_pg_backup.yml"
      end
  end
  
  config.vm.define "pgmaster" do |pgmaster|
    pgmaster.vm.network "private_network", ip: "192.168.50.10",
    virtualbox__intnet: "net1"
    pgmaster.vm.hostname = "pgmaster"
    config.vm.provision "ansible" do |ansible|
        ansible.playbook = "play_pg_master.yml"
      end
  end

  config.vm.define "pgslave" do |pgslave|
    pgslave.vm.network "private_network", ip: "192.168.50.11",
    virtualbox__intnet: "net1"
    pgslave.vm.hostname = "pgslave"
	  config.vm.provision "ansible" do |ansible|
        ansible.playbook = "play_pg_slave.yml"
      end
  end

end