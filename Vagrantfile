# -*- mode: ruby -*-
# vi: set ft=ruby :

# Defaults
defaults = {
  "vm_box" => "centos/7",
  "vm_name" => "VM",
  "vm_num_instances" => 1,
  "enable_serial_logging" => false,
  "sync_folder" => true,
  "vm_gui" => false,
  "vm_memory" => 2048,
  "vm_cpus" => 2,
  "forwarded_ports" => {}
}

require 'fileutils'
require 'yaml'

Vagrant.require_version ">= 1.6.0"

# Load YAML configuration file
dir = File.dirname(File.expand_path(__FILE__))
config_file = "#{dir}/vagrant.yml"
if File.exist?(config_file)
  vconfig = YAML::load_file(config_file)
  vconfig = defaults.merge(vconfig)
end

# https://rubygems.org/gems/to_bool
class String
  def to_bool
    %w{ 1 true yes t }.include? self.downcase
  end
end

class Integer
  def to_bool
    self == 1
  end
end


class TrueClass
  def to_bool
    self
  end
end

class Object
  def to_bool
    false
  end
end

# Get all variables
$vm_box = vconfig["vm_box"].to_s
$vm_name = vconfig["vm_name"].to_s
$vm_num_instances = vconfig["vm_num_instances"].to_i
$enable_serial_logging = vconfig["enable_serial_logging"].to_bool
$sync_folder = vconfig["sync_folder"].to_bool
$vm_gui = vconfig["vm_gui"].to_bool
$vm_memory = vconfig["vm_memory"].to_i
$vm_cpus = vconfig["vm_cpus"].to_i
$forwarded_ports = vconfig["forwarded_ports"]


Vagrant.configure("2") do |config|
  # always use Vagrants insecure key
  config.ssh.insert_key = false

  # This explicitly sets the order that vagrant will use by default
  # if no --provider given
  config.vm.provider "virtualbox"
  config.vm.provider "libvirt"

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = $vm_box

  # enable hostmanager
  config.hostmanager.enabled = true

  # configure the host's /etc/hosts
  config.hostmanager.manage_host = true

  (1..$vm_num_instances).each do |i|
    config.vm.define vm_name = "%s-%02d" % [$vm_name, i] do |config|
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

      # Share an additional folder to the guest VM. The first argument is
      # the path on the host to the actual folder. The second argument is
      # the path on the guest to mount the folder. And the optional third
      # argument is a set of non-required options.
      # config.vm.synced_folder "../data", "/vagrant_data"
      config.vm.synced_folder ".", "/vagrant", disabled: !$sync_folder

      # foward Docker registry port to host for node 01
      if i == 1
        config.vm.network :forwarded_port, guest: 5000, host: 5000
      end

      config.vm.provider :virtualbox do |vb|
        vb.gui = $vm_gui
        vb.memory = $vm_memory
        vb.cpus = $vm_cpus
      end

      ip = "192.168.70.#{i+100}"
      config.vm.network :private_network, ip: ip

    end
  end
end
