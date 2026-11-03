SUBNET = "192.168.56"
NODES = {
  "web01" => [21, 768, 1],
  "db01" => [22, 768, 1],
  "mon01" => [23, 1536, 2]
}

Vagrant.configure("2") do |config|

  config.vm.box = "bento/ubuntu-24.04"
  config.vm.box_check_update = false
  config.vm.boot_timeout = 600
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.linked_clone = true
  end

  config.vm.define "ctrl" do |m|
    m.vm.hostname = "ctrl"
    m.vm.network "private_network", ip: "#{SUBNET}.20"

    m.vm.provider "virtualbox" do |vb|
      vb.name = "asb-ctrl"
      vb.memory = 1024
      vb.cpus = 2
    end
  end

  NODES.each do |nom, (ip,ram,vcpu)|
    config.vm.define nom do |m|
      m.vm.hostname = nom
      m.vm.network "private_network", ip: "#{SUBNET}.#{ip}"
      m.vm.provider "virtualbox" do |vb|
        vb.name = "asb-#{nom}"
        vb.memory = ram
        vb.cpus = vcpu
      end
    end
  end

end
