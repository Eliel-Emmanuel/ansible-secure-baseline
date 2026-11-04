require "fileutils"

KEY_DIR = File.join(__dir__, "vagrant", "keys")
KEY_PATH = File.join(KEY_DIR, "lab_ed25519")

unless File.exist?(KEY_PATH)
  FileUtils.mkdir_p(KEY_DIR)
  ok = system("ssh-keygen", "-q", "-t", "ed25519", "-N", "", "-C", "ansible@lab", "-f", KEY_PATH)
  abort("Échec de ssh-keygen : vérifiez que le client OpenSSH de Windows est installé.") unless ok
  if Vagrant::Util::Platform.windows?
    system("icacls", KEY_PATH, "/inheritance:r", "/grant:r", "#{ENV['USERNAME']}:R")
  end
end

PUBKEY = File.read("#{KEY_PATH}.pub").strip

HOSTS = <<~TEXTE
  192.168.56.20 ctrl.lab.test ctrl
  192.168.56.21 web01.lab.test web01
  192.168.56.22 db01.lab.test db01
  192.168.56.23 mon01.lab.test mon01
TEXTE

COMMON = <<~SHELL
  set -euo pipefail
  sed -i '/lab\\.test/d' /etc/hosts
  echo "$LAB_HOSTS" >> /etc/hosts
  timedatectl set-timezone Africa/Lagos
SHELL

MANAGED = <<~SHELL
  set -euo pipefail
  id ansible >/dev/null 2>&1 || useradd -m -s /bin/bash ansible
  install -d -m 700 -o ansible -g ansible /home/ansible/.ssh
  echo "$LAB_PUBKEY" > /home/ansible/.ssh/authorized_keys
  chown ansible:ansible /home/ansible/.ssh/authorized_keys
  chmod 600 /home/ansible/.ssh/authorized_keys
  echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible
  chmod 440 /etc/sudoers.d/ansible
  visudo -cf /etc/sudoers.d/ansible
SHELL

CTRL_ROOT = <<~SHELL
  set -euo pipefail
  export DEBIAN_FRONTEND=noninteractive
  apt-get update -q
  apt-get install -y -q git make jq python3-venv python3-pip docker.io
  usermod -aG docker vagrant
SHELL

CTRL_USER = <<~SHELL
  set -euo pipefail
  install -d -m 700 ~/.ssh
  install -m 600 ~/projet/vagrant/keys/lab_ed25519 ~/.ssh/lab_ed25519
  [ -d ~/.venvs/ansible ] || python3 -m venv ~/.venvs/ansible
  ~/.venvs/ansible/bin/pip install -q --upgrade pip
  ~/.venvs/ansible/bin/pip install -q -r ~/projet/requirements-dev.txt
  cd ~/projet && ~/.venvs/ansible/bin/ansible-galaxy collection install -r requirements.yml
  grep -q 'venvs/ansible' ~/.bashrc || \\
    echo 'source ~/.venvs/ansible/bin/activate && cd ~/projet' >> ~/.bashrc
SHELL

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
  config.vm.provision "hosts", type: "shell", inline: COMMON, env: { "LAB_HOSTS" => HOSTS }

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.linked_clone = true
  end

  config.vm.define "ctrl" do |m|
    m.vm.hostname = "ctrl"
    m.vm.network "private_network", ip: "#{SUBNET}.20"
    m.vm.synced_folder ".", "/home/vagrant/projet",
      owner: "vagrant", group: "vagrant",
      mount_options: ["dmode=755", "fmode=644"]

    m.vm.provision "outils", type: "shell", inline: CTRL_ROOT
    m.vm.provision "ansible-env", type: "shell", privileged: false, inline: CTRL_USER

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
      m.vm.provision "compte-ansible", type: "shell", inline: MANAGED, env: { "LAB_PUBKEY" => PUBKEY }
      m.vm.provider "virtualbox" do |vb|
        vb.name = "asb-#{nom}"
        vb.memory = ram
        vb.cpus = vcpu
      end
    end
  end

end
