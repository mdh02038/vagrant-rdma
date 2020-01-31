# -*- mode: ruby -*-
# vi: set ft=ruby :


#
# configuration parameters
#Box = "ubuntu/eoan64"
Box = "generic/ubuntu1910"
#Box = "ubuntu/bionic64"
BuildKernel = true
NodeCount = 2

#
# local variables
#
masterGroup = []
hostVars = {}


#
# create apt cache for each vm
#
def local_cache(box_name, hostname)
  cache_dir = File.join( File.expand_path("./vagrant_apt"),
    'cache',
    'apt',
    box_name,
    hostname
    )

  partial_dir = File.join(cache_dir, 'partial')
  FileUtils.mkdir_p(partial_dir) unless File.exists? partial_dir
  cache_dir
end


# create cache directory
cacheDir='./cache'
FileUtils.mkdir_p(cacheDir) unless File.exists? cacheDir 
#
# create masters and workers
#
Vagrant.configure("2") do |config|
  config.vm.box = Box
  # add entries to hostfile for each vm
  config.hostmanager.enable = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true
  (1..NodeCount).each do |id|
    name = "rd#{id}"
    masterGroup.push( name )

    config.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--nictype1", "virtio"]        
        vb.customize ["modifyvm", :id, "--nictype2", "virtio"]        
    end
     
    isPrimary = id==1
    myIp = "172.18.0.#{1+id}"
    hostVars[name] = { 
        "private_ip" => myIp, 
        "ssh_port" => "22", 
        "ssh_host" => myIp
    }

    config.vm.define name, primary: isPrimary do |node|
      cache_dir = local_cache(config.vm.box,name)
      node.vm.synced_folder cache_dir, "/var/cache/apt/archives/"
      config.vm.synced_folder ".", "/vagrant", owner: "vagrant", type: "virtualbox"
      node.vm.hostname = name
      if BuildKernel 
          node.disksize.size = '60GB'
      end
      node.vm.network :private_network, ip: myIp
      node.vm.network :forwarded_port, guest: 22, host: 2000+(21+id), id: "ssh"
      node.vm.provider :virtualbox do |v|
        v.linked_clone = true
        if BuildKernel
            v.memory = 4*1024 
            v.cpus = 2
        else
            v.memory = 1*1024 
            v.cpus = 1
        end
        v.name = name
      end
      node.vm.provision "shell", inline: <<-SHELL
         apt-get update
         apt-get --no-install-recommends install virtualbox-guest-utils
         apt-get install -y python3 aptitude python3-pip python3-pexpect
      SHELL
      if id == NodeCount
        node.vm.provision "ansible" do |ansible|
           ansible.limit = "all"
           ansible.playbook = "vagrant_playbook.yml"
#           ansible.verbose = "-vvvv"
#           ansible.verbose = "true"
           ansible.compatibility_mode = "2.0"
           ansible.host_vars = hostVars
           ansible.extra_vars = {
             "ansible_python_interpreter" => "python3",
             "private_interface" => "eth1",
             "buildKernel" => BuildKernel
           }
           ansible.groups = {
             "Master" => masterGroup,
           }
        end
      end
    end
  end
end
