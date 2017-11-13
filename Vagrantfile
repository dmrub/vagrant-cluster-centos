# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'
require 'yaml'

Vagrant.require_version ">= 1.6.0"

# Defaults for config options defined in CONFIG
$box_name = "dmrub/centos7" # "centos/7"
$num_instances = 3
$instance_name_prefix = "kube"
$enable_serial_logging = false
$share_home = false
$vm_gui = false
$vm_memory = 2048
$vm_cpus = 2
$forwarded_ports = {}

# dir = File.dirname(File.expand_path(__FILE__))
# if File.exist?("#{dir}/config.yml")
#   vconfig = YAML::load_file("#{dir}/config.yml")
#   $num_instances = vconfig["num_instances"]
#   $instance_name_prefix = vconfig["instance_name_prefix"]
#   $enable_serial_logging = false
#   $share_home = false
#   $vm_gui = false
#   $vm_memory = 3072
#   $vm_cpus = 2
#   $forwarded_ports = {}

# end


# Attempt to apply the deprecated environment variable NUM_INSTANCES to
# $num_instances while allowing config.rb to override it
if ENV["NUM_INSTANCES"].to_i > 0 && ENV["NUM_INSTANCES"]
  $num_instances = ENV["NUM_INSTANCES"].to_i
end

# Use old vb_xxx config variables when set
def vm_gui
  $vb_gui.nil? ? $vm_gui : $vb_gui
end

def vm_memory
  $vb_memory.nil? ? $vm_memory : $vb_memory
end

def vm_cpus
  $vb_cpus.nil? ? $vm_cpus : $vb_cpus
end

Vagrant.configure("2") do |config|
  # always use Vagrants insecure key
  config.ssh.insert_key = false

  # This explicitly sets the order that vagrant will use by default if no --provider given
  config.vm.provider "virtualbox"
  config.vm.provider "libvirt"

  config.vm.box = $box_name

  # enable hostmanager
  config.hostmanager.enabled = true

  # configure the host's /etc/hosts
  config.hostmanager.manage_host = true

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "%s-%02d" % [$instance_name_prefix, i] do |config|
      config.vm.hostname = vm_name

      if $enable_serial_logging
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
        FileUtils.touch(serialFile)

        # https://serverfault.com/questions/453185/vagrant-virtualbox-dns-10-0-2-3-not-working
        # VBoxManage modifyvm "VM name" --natdnsproxy1 on
        # VBoxManage modifyvm "VM name" --natdnshostresolver1 on
        config.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
          vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        end
      end

      # foward Docker registry port to host for node 01
      if i == 1
        config.vm.network :forwarded_port, guest: 5000, host: 5000
      end

      config.vm.provider :virtualbox do |vb|
        vb.gui = vm_gui
        vb.memory = vm_memory
        vb.cpus = vm_cpus
      end

      ip = "192.168.70.#{i+100}"
      config.vm.network :private_network, ip: ip

    end
  end
end
