# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define "factory" do |factory|
    factory.vm.box = "geerlingguy/ubuntu1804"
    factory.ssh.insert_key = false
    factory.vm.hostname = 'factory'
    factory.vm.network :private_network, ip: "192.168.0.43"
    factory.vm.synced_folder "~/apps", "/var/www/vhosts"
    factory.vm.synced_folder "~/.ssh", "/home/vagrant/.ssh"
    factory.vm.synced_folder "~/dev_cert", "/etc/apache2/ssl"
    factory.vm.synced_folder "~/.aws", "/home/vagrant/.aws"

    factory.vm.provider :virtualbox do |v|
      v.name = "factory"
      v.memory = 2048
      v.cpus = 2
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ["modifyvm", :id, "--ioapic", "on"]
    end

    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "main.yml"
      ansible.become = true
      #ansible.raw_arguments  = "--ask-vault-pass"
      #ansible.raw_arguments  = "--vault-password-file=~/devops/vault_pw/ansible_vault_pass.yml"
      #ansible.tags = "update"
      #ansible.skip_tags =
    end
  end
end
