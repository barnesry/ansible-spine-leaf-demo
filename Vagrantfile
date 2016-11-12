# -*- mode: ruby -*-
# vi: set ft=ruby :


# Vagrant Toplogy
#       +---------+              +---------+
#       |         |              |         |
#       | SPINE01 |              | SPINE02 |
#       |         |              |         |
#       +---------+              +---------+
#    ge-0/0/1|     X  ge-0/0/2  X     | ge-0/0/1
#            |       X        X       |
#            |         X    X         |
#            |           XX           |
#            |          X  X          |
#    ge-0/0/1|        X      X        | ge-0/0/1
#       +---------+ X          X +---------+
#       |         |   ge-0/0/2   |         |
#       | SPINE01 |              | SPINE02 |
#       |         |              |         |
#       +---------+              +---------+




VAGRANTFILE_API_VERSION = "2"
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|


    config.ssh.insert_key = false
    config.ssh.username = "vagrant"

    # Servers for traffic generation
    (1..2).each do |id|

        srv_name  = ( "srv" + id.to_s ).to_sym
        
        # set the network starting at 100 + id (ie. id of 1 = 101.101.101.0)
        net_id = 100 + id
        net = "#{net_id}."*3 + "0"              # eg. 101.101.101.0
        router_ip = "#{net_id}."*3 + "1"        # eg. 101.101.101.1
        server_ip = "#{net_id}."*3 + "2"        # eg. 101.101.101.2

        ##########################
        ## Server          #######
        ##########################
        config.vm.define srv_name do |srv|
            srv.vm.box = "robwc/minitrusty64"
            srv.vm.hostname = "server-leaf#{id}"

            srv.vm.provider "virtualbox" do |v|
                v.name = srv.vm.hostname
                v.memory = 512
            end

            srv.vm.network 'private_network', ip: server_ip, virtualbox__intnet: "server#{id}_to_leaf#{id}"
            srv.ssh.insert_key = true
	        srv.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

	    srv.vm.provision "shell",
            inline: "sudo route add -net #{net} netmask 255.255.255.0 gw #{router_ip}
                sudo apt-get -y update
                sudo apt-get -y install traceroute curl telnet
                sudo apt-get clean"
       end
    end

    ## SPINE ROUTERS ##
    (1..2).each do |id|

        re_name  = ( "SPINE0" + id.to_s ).to_sym

        if id == 2
            alt_leaf_id = 1
        elsif id == 1
            alt_leaf_id = 2 
        end

        ##########################
        ## Routing Engine  #######
        ##########################
        config.vm.define re_name do |vsrx|
            vsrx.vm.hostname = re_name
            # vsrx.vm.box = 'juniper/ffp-12.1X47-D20.7'
            vsrx.vm.box = 'juniper/ffp-12.1X47-D15.4-packetmode'

            vsrx.vm.provider "virtualbox" do |v|
                v.name = "vagrant-" + vsrx.vm.hostname.to_s
                v.memory = 1024
                v.cpus = 2
            end

            # DO NOT REMOVE / NO VMtools installed
            vsrx.vm.synced_folder '.', '/vagrant', disabled: true

            # Management port
            # vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "vboxnet0"

            # Dataplane ports
            # ge-0/0/1
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "spine#{id}_to_leaf#{id}"
            # ge-0/0/2
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "spine#{id}_to_leaf#{alt_leaf_id}"
        end
    end


    ## LEAF ROUTERS ##
    (1..2).each do |id|

        re_name  = ( "LEAF0" + id.to_s ).to_sym

        if id == 2
            alt_spine_id = 1
        elsif id == 1
            alt_spine_id = 2
        end

        ##########################
        ## Routing Engine  #######
        ##########################
        config.vm.define re_name do |vsrx|
            vsrx.vm.hostname = re_name
            # vsrx.vm.box = 'juniper/ffp-12.1X47-D20.7'
            vsrx.vm.box = 'juniper/ffp-12.1X47-D15.4-packetmode'

            vsrx.vm.provider "virtualbox" do |v|
                v.name = "vagrant-" + vsrx.vm.hostname.to_s
                v.memory = 1024
                v.cpus = 2
            end

            # DO NOT REMOVE / NO VMtools installed
            vsrx.vm.synced_folder '.', '/vagrant', disabled: true

            # Management port
            # vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "vboxnet0"

            # Dataplane ports
            # ge-0/0/1
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "spine#{id}_to_leaf#{id}"
            # ge-0/0/2
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "spine#{alt_spine_id}_to_leaf#{id}"
            # ge-0/0/3
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "server#{id}_to_leaf#{id}"
        end
    end


 
    ##############################
    ## Box provisioning    #######
    ##############################
    if !Vagrant::Util::Platform.windows?
        config.vm.provision "ansible" do |ansible|
            ansible.groups = {
                "spine" => [ "SPINE01", "SPINE02" ],
                "leaf" => [ "LEAF01", "LEAF02" ],
                "server"  => [ "server1", "server2" ],
                "juniper:children" => [ "spine", "leaf" ],
                "all:children" => [ "juniper", "servers" ],
                "juniper:vars" => {  "ansible_ssh_user" => "vagrant",
                                     "ansible_ssh_password" => "junos123",
                                     "junos_ssh_user" => "root",
                                     "junos_pwd_clear" => "junos123",
                                     "netconf_port" =>  "830",
                                     "mgmt_port"    =>  "em0",
                                     "mgmt_sub_mask"    =>  "23",
                                     "mgmt_sub_gw"      =>  "0.0.0.0",
                                     "junos_host"       => "0.0.0.0" 
                                  }   
            }
            # we'll use the dynamic inventory in .vagrant/provisioners/anisble/inventory instead'
            # ansible.inventory_path = "devices.ini"
            ansible.verbose = "v"
            ansible.playbook = "pb.config.all.commit.yaml"
            ansible.force_remote_user = "vagrant"
        end
    end
end
