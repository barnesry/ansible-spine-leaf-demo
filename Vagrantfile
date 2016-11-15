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
       
    ##########################
    ## Services VM     #######
    ##########################
    config.vm.define "toolsVM" do |srv|
        srv.vm.box = "robwc/minitrusty64"
        srv.vm.hostname = "toolsVM"

        srv.vm.provider "virtualbox" do |v|
            v.name = "vagrant-toolsVM"
            v.memory = 1024
        end

        # vagrant synced folder
        srv.vm.synced_folder '.', '/vagrant', disabled: false

        # Network Interfaces
        # eth0 will be public bridge (NAT)
        # eth1 on management network
        srv.vm.network 'private_network', type: "dhcp"
        
        srv.ssh.insert_key = false
        srv.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

        srv.vm.provision "shell",
            inline: "wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
                sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
                sudo apt-get install -y software-properties-common
                sudo apt-add-repository -y ppa:ansible/ansible
                sudo apt-get update
                sudo apt-get upgrade
                sudo apt-get install -y python python-dev python-pip iputils-ping
                sudo apt-get install -y libxml2-dev libxslt-dev libssl-dev libffi-dev
                sudo pip install --upgrade pip
                sudo pip install junos-eznc
                sudo apt-get install -y jenkins
                sudo apt-get install -y ansible
                sudo ansible-galaxy install Juniper.junos
                sudo curl https://raw.githubusercontent.com/ansible/ansible/devel/examples/ansible.cfg > /etc/ansible/ansible.cfg
                sudo apt-get -y install traceroute curl telnet paris-traceroute mtr
                sudo apt-get autoremove
                sudo apt-get clean
                mkdir -p /home/vagrant/ansible/group_vars
                mkdir -p /home/vagrant/ansible/host_vars
                mkdir -p /home/vagrant/ansible/roles/tasks
                mkdir -p /home/vagrant/ansible/roles/handlers
                mkdir -p /home/vagrant/ansible/roles/templates
                mkdir -p /home/vagrant/ansible/roles/files
                mkdir -p /home/vagrant/ansible/roles/vars
                mkdir -p /home/vagrant/ansible/roles/defaults
                mkdir -p /home/vagrant/ansible/roles/meta
                mkdir -p /home/vagrant/ansible/library
                mkdir -p /home/vagrant/ansible/filter_plugins
                touch /home/vagrant/ansible/inventory.ini
                touch /home/vagrant/ansible/pb.master.yaml
                chown -R vagrant /home/vagrant/ansible
                chgrp -R vagrant /home/vagrant/ansible"
        end

    
    
    # client servers for traffic generation connected to leaf nodes
    (1..2).each do |id|

        srv_name  = ( "srv" + id.to_s ).to_sym
        
        # set the network starting at 100 + id (ie. id of 1 = 101.101.101.0)
        net_id = 100 + id
        net = "#{net_id}."*3 + "0"              # eg. 102.102.102.0
        router_ip = "#{net_id}."*3 + "1"        # eg. 102.102.102.1
        server_ip = "#{net_id}."*3 + "2"        # eg. 102.102.102.2

        if id == 1
            remote_net = "#{net_id+1}."*3 + "0" # eg. 101.101.101.0
            remote_host = "#{net_id+1}."*3 + "1" # eg. 101.101.101.1
        elsif id == 2
            remote_net = "#{net_id-1}."*3 + "0"  # eg. 102.102.102.0
            remote_host = "#{net_id-1}."*3 + "1"  # eg. 102.102.102.1
        end    


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
            inline: "sudo route add -net #{remote_net} netmask 255.255.255.0 gw #{router_ip}
                sudo route add -net 171.0.0.0 netmask 255.255.255.0 gw #{router_ip}
                sudo apt-get -y update
                sudo apt-get -y install traceroute curl telnet paris-traceroute mtr
                sudo apt-get autoremove
                sudo apt-get clean"
            if id == 1
                srv.vm.provision "shell",
                    inline: "sudo -- sh -c 'echo #{remote_host} srv2' >> /etc/hosts"
            elsif id ==2
            srv.vm.provision "shell",
                    inline: "sudo -- sh -c 'echo #{remote_host} srv1' >> /etc/hosts"
            end
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

            # Dataplane ports
            # ge-0/0/1
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "spine#{id}_to_leaf#{id}"
            # ge-0/0/2
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', virtualbox__intnet: "spine#{id}_to_leaf#{alt_leaf_id}"
            # Management port - ge-0/0/3
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', type: "dhcp"
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
            # Management port - ge-0/0/4
            vsrx.vm.network 'private_network', auto_config: false, nic_type: 'virtio', type: "dhcp"
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
                "leaf:vars" => { "mgmt_port" => "ge-0/0/4" },
                "spine:vars" => { "mgmt_port" => "ge-0/0/3" },
                "juniper:vars" => {  "junos_ssh_user" => "root",
                                     "junos_pwd_clear" => "junos123",
                                     "netconf_port" =>  "830",
                                     "mgmt_sub_mask"    =>  "24",
                                     "mgmt_sub_gw"      =>  "0.0.0.0" 
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

                # "leaf" => [ "LEAF01" => { "junos_host" => "192.168.56.101",
                #                           "loopback_ip" => "1.1.1.1",
                #                           "mgmt_port" => "ge-0/0/3" 
                #                         },
                #             "LEAF02" => { "junos_host" => "192.168.56.102",
                #                           "loopback_ip" => "1.1.1.2",
                #                           "mgmt_port" => "ge-0/0/3" 
                #                         }
                #              ],
