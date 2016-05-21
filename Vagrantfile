# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "debian/jessie64"

  config.vm.network "forwarded_port", guest: 22, host: 10022
  config.vm.network "forwarded_port", guest: 25, host: 10025
  config.vm.network "forwarded_port", guest: 80, host: 10080
  config.vm.network "forwarded_port", guest: 443, host: 10443
  config.vm.network "forwarded_port", guest: 993, host: 10993

  config.vm.provision "shell", inline: <<-SHELL
    mkdir -p /root/.ssh || true
    chmod 700 /root/.ssh
    echo "#{File.read(File.join(Dir.home, '.ssh', 'id_rsa.pub'))}" > /root/.ssh/authorized_keys
    chmod 600 /root/.ssh/authorized_keys
  SHELL
end
