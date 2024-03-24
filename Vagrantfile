# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

    config.vm.box = "centos/7"
    
    config.vm.provision "ansible" do |ansible|
    #ansible.verbose = "vvv"
    ansible.playbook = "provision.yml"
    #ansible.become = true
    ansible.limit = "all"
    ansible.raw_arguments = Shellwords.shellsplit(ENV['ANSIBLE_ARGS']) if ENV['ANSIBLE_ARGS']
    end

    config.vm.define "ipaserver" do |server|
        server.vm.host_name = 'ipa3.otus.lan'
        server.vm.network :private_network, ip: "192.168.57.23"

        server.vm.network "forwarded_port", guest: 80, host: 80
        server.vm.network "forwarded_port", guest: 443, host: 443

        server.vm.provider "virtualbox" do |vb|
            vb.memory = 2048
            vb.cpus = "2"
        end
    end
    
    config.vm.define "ipaclient" do |client|
        client.vm.host_name = 'client3.otus.lan'
        client.vm.network :private_network, ip: "192.168.57.13"
    end    
end
