# Created by Jonas Rosland, @virtualswede & Matt Cowger, @mcowger
# Many thanks to this post by James Carr: http://blog.james-carr.org/2013/03/17/dynamic-vagrant-nodes/
# Extended by cebruns for CoprHD All-In-One Vagrant Setup

########################################################
#
# Global Settings for Environment
#
########################################################
network = "192.168.100"
domain = 'aio.local'
# Change to ubuntu if you want the default to be Ubuntu-based VM/CoprHD
coprhd_vm = 'opensuse'

script_proxy_args = ""
# Check if we are currently behind proxy
# We will pass into build/provision scripts if set
if ENV["http_proxy"] || ENV["https_proxy"]
  if !(Vagrant.has_plugin?("vagrant-proxyconf"))
    abort "Env Proxy set but vagrant-proxyconf not installed. Fix with: vagrant plugin install vagrant-proxyconf"
   end
   # Remove http and https from proxy setting
   temp = ENV["http_proxy"].dup
   temp.slice! "http://"
   http_proxy, http_proxy_port = temp.split(":")
   script_proxy_args = " --proxy #{http_proxy} --port #{http_proxy_port}"

   # Some proxies use http or https as secure proxy, handle both
   temp = ENV["https_proxy"].dup
   temp =~ /https*:\/\/(.*)/
   https_proxy, https_proxy_port = $1.split(":")
   script_proxy_args += " --secure_proxy #{https_proxy} --secure_port #{https_proxy_port}"
   script_proxy_args += " --secure_proxy #{https_proxy} --secure_port #{https_proxy_port}"
end

########################################################
#
# CoprHD Settings
#
########################################################
if ARGV.include? 'deb_coprhd'
    coprhd_vm = 'ubuntu'
    puts "Ubuntu Chosen for CoprHD VM"
elsif ARGV.include? 'opensuse_coprhd'
    coprhd_vm = 'opensuse'
    puts "OpenSuse Chosen for CoprHD VM"
end
ch_node_ip = "#{network}.11"
ch_virtual_ip = "#{network}.10"
ch_gw_ip = "#{network}.1"
ch_vagrantbox = "vchrisb/openSUSE-13.2_64"
ch_vagrantboxurl = "https://atlas.hashicorp.com/vchrisb/boxes/openSUSE-13.2_64/versions/0.1.3/providers/virtualbox.box"
deb_ch_vagrantbox = "ubuntu/trusty64"
deb_ch_vagrantboxurl = "https://atlas.hashicorp.com/ubuntu/boxes/trusty64/versions/20160120.0.1/providers/virtualbox.box"
# Default is to build CoprHD when Vagrant provisioning new VM
build = true
# Default is to not include the SMI-S Simulator for backends, we use ScaleIO
smis_simulator = false

########################################################
#
# ScaleIO Settings
#
########################################################
# ScaleIO vagrant box
sio_vagrantbox="centos_6.5"

# ScaleIO vagrant box url
sio_vagrantboxurl="https://github.com/2creatives/vagrant-centos/releases/download/v6.5.3/centos65-x86_64-20140116.box"

# scaleio admin password
sio_password="Scaleio123"

# add your nodes here
sio_nodes = ['tb', 'mdm1', 'mdm2']

clusterip = "#{network}.20"
tbip = "#{network}.21"
firstmdmip = "#{network}.22"
secondmdmip = "#{network}.23"

# Install ScaleIO cluster automatically or Installation Manager (IM) only
# If True a fully working ScaleIO cluster is installed.
# False means only IM is installed on node MDM1.
clusterinstall = "True"

# version of installation package
version = "1.32-402.1"

#OS Version of package
os="el6"

# installation folder
siinstall = "/opt/scaleio/siinstall"

# packages folder
packages = "/opt/scaleio/siinstall/ECS/packages"
# package name, was ecs for 1.21, is now EMC-ScaleIO from 1.30
packagename = "EMC-ScaleIO"

# fake device
device = "/home/vagrant/scaleio1"

# loop through the nodes and set hostname
scaleio_nodes = []
sio_nodes.each { |node_name|
  (1..1).each {|n|
    scaleio_nodes << {:hostname => "#{node_name}"}
  }
}

########################################################
#
# Launch the VMs
#
########################################################
Vagrant.configure("2") do |config|

  # If Proxy is set when provisioning, we set it permanently in each VM
  # If Proxy is not set when provisioning, we won't set it
  if Vagrant.has_plugin?("vagrant-proxyconf")
    if ENV["http_proxy"]
      config.proxy.http    = ENV["http_proxy"]
    end
    if ENV["https_proxy"]
      config.proxy.https   = ENV["https_proxy"]
    end
    if ENV["ftp_proxy"]
      config.proxy.ftp     = ENV["ftp_proxy"]
    end
    if ENV["no_proxy"]
      config.proxy.no_proxy = ENV["no_proxy"]
    end
  end

  # Enable caching to speed up package installation for second run
  # vagrant plugin install vagrant-cachier
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end

########################################################
#
# Launch ScaleIO
#
########################################################
  scaleio_nodes.each do |node|
    config.vm.define node[:hostname] do |node_config|
      node_config.vm.box = "#{sio_vagrantbox}"
      node_config.vm.box_url = "#{sio_vagrantboxurl}"
      node_config.vm.host_name = "#{node[:hostname]}.#{domain}"
      node_config.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "1024"]
        vb.customize ["modifyvm", :id, "--name", node[:hostname]]
      end

      if node[:hostname] == "tb"
        node_config.vm.network "private_network", ip: "#{tbip}"
        node_config.vm.provision "shell" do |s|
          s.path = "scripts/tb.sh"
          s.args   = "-o #{os} -v #{version} -n #{packagename} -d #{device} -f #{firstmdmip} -s #{secondmdmip} -i #{siinstall} -c #{clusterinstall}"
        end
        # Setup ntpdate crontab
        node_config.vm.provision "shell" do |s|
          s.path = "scripts/crontab.sh"
        end
      end

      if node[:hostname] == "mdm1"
        node_config.vm.network "private_network", ip: "#{firstmdmip}"
        node_config.vm.network "forwarded_port", guest: 6611, host: 6611
        node_config.vm.provision "shell" do |s|
          s.path = "scripts/mdm1.sh"
          s.args   = "-o #{os} -v #{version} -n #{packagename} -d #{device} -f #{firstmdmip} -s #{secondmdmip} -i #{siinstall} -p #{sio_password} -c #{clusterinstall}"
        end
        # Setup ntpdate crontab
        node_config.vm.provision "shell" do |s|
          s.path = "scripts/crontab.sh"
        end
      end

      if node[:hostname] == "mdm2"
        node_config.vm.network "private_network", ip: "#{secondmdmip}"
        node_config.vm.provision "shell" do |s|
          s.path = "scripts/mdm2.sh"
          s.args   = "-o #{os} -v #{version} -n #{packagename} -d #{device} -f #{firstmdmip} -s #{secondmdmip} -i #{siinstall} -t #{tbip} -p #{sio_password} -c #{clusterinstall}"
        end
        # Setup ntpdate crontab
        node_config.vm.provision "shell" do |s|
          s.path = "scripts/crontab.sh"
        end
      end
    end
  end

########################################################
#
# Launch CoprHD
#
########################################################
  if coprhd_vm == 'opensuse' then
  puts "Default (opensuse) CoprHD Instance"
  config.vm.define "coprhd" do |coprhd|
     coprhd.vm.box = "#{ch_vagrantbox}"
     coprhd.vm.box_url = "#{ch_vagrantboxurl}"
     coprhd.vm.host_name = "coprhd1.#{domain}"
     coprhd.vm.network "private_network", ip: "#{ch_node_ip}"

     # configure virtualbox provider
     coprhd.vm.provider "virtualbox" do |v|
         v.gui = false
         v.name = "CoprHD"
         v.memory = 3000
         v.cpus = 4
     end

     # Setup Swap space
     coprhd.vm.provision "shell" do |s|
      s.path = "scripts/swap.sh"
     end

     # install necessary packages
     coprhd.vm.provision "shell" do |s|
      s.path = "scripts/packages.sh"
      s.args   = "--build #{build}"
     end

     # download, patch and build nginx
     coprhd.vm.provision "shell" do |s|
      s.path = "scripts/nginx.sh"
      s.args = ""
     end

     # create CoprHD configuration file
     coprhd.vm.provision "shell" do |s|
      s.path = "scripts/config.sh"
      s.args   = "--node_ip #{ch_node_ip} --virtual_ip #{ch_virtual_ip} --gw_ip #{ch_gw_ip} --node_count 1 --node_id vipr1"
     end

     # download and compile CoprHD from sources
     coprhd.vm.provision "shell" do |s|
      s.path = "scripts/build.sh"
      s.args   = "--build #{build}"
      s.args  += script_proxy_args
     end

      # Setup ntpdate crontab
      coprhd.vm.provision "shell" do |s|
        s.path = "scripts/crontab.sh"
        s.privileged = false
      end

     # install CoprHD RPM
     coprhd.vm.provision "shell" do |s|
      s.path = "scripts/install.sh"
      s.args   = "--virtual_ip #{ch_virtual_ip}"
     end

     # Grab CoprHD CLI Scripts and Patch Auth Module
     coprhd.vm.provision "shell" do |s|
      s.path = "scripts/coprhd_cli.sh"
      s.args = "-s #{smis_simulator}"
     end

     coprhd.vm.provision "shell" do |s|
      s.path = "scripts/banner.sh"
      s.args   = "--virtual_ip #{ch_virtual_ip}"
     end
  end
  end
########################################################
#
# Launch Ubuntu CoprHD
#
########################################################
  if coprhd_vm == 'ubuntu' then
  puts "Ubuntu CoprHD"
  config.vm.define "deb_coprhd" do |deb_coprhd|
     deb_coprhd.vm.box = "#{deb_ch_vagrantbox}"
#    deb_coprhd.vm.box_url = "#{deb_ch_vagrantboxurl}"
     deb_coprhd.vm.host_name = "coprhd1.#{domain}"
     deb_coprhd.vm.network "private_network", ip: "#{ch_node_ip}"

     # configure virtualbox provider
     deb_coprhd.vm.provider "virtualbox" do |v|
         v.gui = false
         v.name = "UbuntuCoprHD"
         v.memory = 3000
         v.cpus = 4
     end

     # install dependencies and CoprHD specific env
     deb_coprhd.vm.provision "shell" do |s|
      s.path = "scripts/deb-inst-deps.sh"
      s.args   = "--build #{build} --node_ip #{ch_node_ip} --virtual_ip #{ch_virtual_ip} --gw_ip #{ch_gw_ip} --node_count 1 --node_id vipr1 "
      s.args  += script_proxy_args
     end

     # upload ubuntu patch
     deb_coprhd.vm.provision "file", source: "coprhd-ubuntu.patch", destination: "/tmp/coprhd-ubuntu.patch"

     # download and compile CoprHD from sources
     deb_coprhd.vm.provision "shell" do |s|
      s.path = "scripts/deb-build-coprhd.sh"
      s.args   = "--build #{build}"
      s.args  += script_proxy_args
     end

      # Setup ntpdate crontab
      deb_coprhd.vm.provision "shell" do |s|
        s.path = "scripts/crontab.sh"
        s.privileged = false
      end

     # install CoprHD
     deb_coprhd.vm.provision "shell" do |s|
      s.path = "scripts/deb-inst-coprhd.sh"
      s.args   = "--virtual_ip #{ch_virtual_ip}"
     end

     # TODO for Ubuntu: Grab CoprHD CLI Scripts and Patch Auth Module
     #deb_coprhd.vm.provision "shell" do |s|
     # s.path = "scripts/coprhd_cli.sh"
     # s.args = "-s #{smis_simulator}"
     #end

     deb_coprhd.vm.provision "shell" do |s|
      s.path = "scripts/banner.sh"
      s.args   = "--virtual_ip #{ch_virtual_ip}"
     end
  end
  end
end
