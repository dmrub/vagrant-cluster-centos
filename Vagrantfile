# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'
require 'yaml'
require 'erb'

Vagrant.require_version ">= 1.6.0"

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

# https://stackoverflow.com/questions/9381553/ruby-merge-nested-hash
class ::Hash
  def deep_merge(second)
      merger = proc { |key, v1, v2| Hash === v1 && Hash === v2 ? v1.merge(v2, &merger) : v2 }
      self.merge(second, &merger)
  end
end

# Defaults

vm_defaults = {
  "box" => "centos/7",
  "name" => "node",
  "name_template" => "<%= '%s-%02d' % [name, index+1] %>",
  "count" => 1,
  "gui" => false,
  "memory" => 2048,
  "cpus" => 2,
  "enable_serial_logging" => false,
  "default_nic_type" => nil,
  "private_network_ip_prefix" => "192.168.70.",
  "private_network_ip_template" => "<%= private_network_ip_prefix %><%= global_index+100 %>",
  "synced_folders" => [],
  "forwarded_ports" => [],
  "forwarded_port_range" => []
}

hostmanager_defaults = {
  "enabled" => true,
  "manage_host" => true
}

# Load YAML configuration file

main_conf = {}
dir = File.dirname(File.expand_path(__FILE__))
config_file = "#{dir}/vagrant.yml"
if File.exist?(config_file)
  main_conf = YAML::load_file(config_file)
end

if main_conf.nil?
  main_conf = {}
end

if !main_conf.key?("vms") || main_conf["vms"].empty?
  puts "WARNING: 'vms' array is empty, no virtual machines configured, creating default !"
  main_conf["vms"] = [vm_defaults]
else
  if !main_conf["vms"].kind_of?(Array)
    puts "Error: vms must be an array"
    abort
  end
end

Vagrant.configure("2") do |config|
  # Global VM index
  global_index = 0

  # always use Vagrants insecure key
  config.ssh.insert_key = false

  # This explicitly sets the order that vagrant will use by default
  # if no --provider given
  config.vm.provider "virtualbox"
  config.vm.provider "libvirt"

  hostmanager_conf = hostmanager_defaults.deep_merge(main_conf.fetch("hostmanager", {}))

  # enable hostmanager
  config.hostmanager.enabled = hostmanager_conf["enabled"].to_bool

  # configure the host's /etc/hosts
  config.hostmanager.manage_host = hostmanager_conf["manage_host"].to_bool

  main_conf["vms"].each do |vm_conf|
    # merge VM configuration with defaults
    vm_conf = vm_defaults.deep_merge(vm_conf)

    # collect per instance settings
    per_instance_vm_conf = {}
    instances_conf = vm_conf.fetch("instances", nil)
    if !instances_conf.nil?
      if !instances_conf.kind_of?(Array)
        puts "Error: 'instances' must be an array"
        abort
      end

      instances_conf.each do |inst|
        if !inst.kind_of?(Hash)
          puts "Error: instance (element of 'instances' array) must be a dictionary"
          abort
        end

        if !inst.key?("index")
          puts "Error: instance (element of 'instances' array) dictionary must contain 'index' key"
          abort
        end

        inst = inst.clone()
        index = inst.delete("index")
        per_instance_vm_conf[index.to_i] = inst
      end
    end

    box = vm_conf["box"].to_s
    name = vm_conf["name"].to_s
    name_template = vm_conf["name_template"].to_s
    count = vm_conf["count"].to_i
    gui = vm_conf["gui"].to_bool
    memory = vm_conf["memory"].to_i
    cpus = vm_conf["cpus"].to_i
    enable_serial_logging = vm_conf["enable_serial_logging"].to_bool
    default_nic_type = vm_conf["default_nic_type"].to_s if vm_conf.key? "default_nic_type"
    private_network_ip_prefix = vm_conf["private_network_ip_prefix"].to_s
    private_network_ip_template = vm_conf["private_network_ip_template"].to_s
    forwarded_ports = vm_conf["forwarded_ports"]
    forwarded_port_range = vm_conf["forwarded_port_range"]
    synced_folders = vm_conf["synced_folders"]

    if !forwarded_ports.kind_of?(Array)
      puts "Error: value of 'forwarded_ports' key must be an array"
      abort
    end

    if !forwarded_port_range.kind_of?(Array)
      puts "Error: value of 'forwarded_port_range' key must be an array"
      abort
    end

    if !synced_folders.kind_of?(Array)
      puts "Error: value of 'synced_folders' key must be an array"
      abort
    end

    (0..count-1).each do |index|
      vars = {
        "box" => box,
        "name" => name,
        "name_template" => name_template,
        "count" => count,
        "index" => index,
        "global_index" => global_index,
        "gui" => gui,
        "memory" => memory,
        "cpus" => cpus,
        "enable_serial_logging" => enable_serial_logging,
        "default_nic_type" => default_nic_type,
        "private_network_ip_prefix" => private_network_ip_prefix,
        "private_network_ip_template" => private_network_ip_template,
        "forwarded_ports" => forwarded_ports,
        "forwarded_port_range" => forwarded_port_range,
        "synced_folders" => synced_folders
      }

      per_instance_vars = per_instance_vm_conf.fetch(index, nil)
      if !per_instance_vars.nil?
        vars = vars.deep_merge(per_instance_vars)
      end

      vm_name_template = ERB.new(vars["name_template"])
      vm_private_network_ip_template = ERB.new(vars["private_network_ip_template"])

      vm_name = vm_name_template.result_with_hash(vars)
      vm_private_ip = vm_private_network_ip_template.result_with_hash(vars)

      puts "VM #{global_index}: #{vm_name}"
      puts "  Private Network IP: #{vm_private_ip}"
      puts "  Box: #{vars['box']}"
      puts "  Index: #{index} / #{count-1}"
      puts "  Global Index: #{global_index}"
      puts "  GUI: #{vars['gui']}"
      puts "  Memory: #{vars['memory']}"
      puts "  CPUs: #{vars['cpus']}"
      puts "  Enable Serial Logging: #{vars['enable_serial_logging']}"
      puts "  Default NIC Type: #{vars['default_nic_type']}"

      global_index += 1

      config.vm.define vm_name do |node|
        node.vm.box = vars['box']
        node.vm.hostname = vm_name

        if vars['enable_serial_logging']
          logdir = File.join(File.dirname(__FILE__), "log")
          FileUtils.mkdir_p(logdir)

          serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
          FileUtils.touch(serialFile)

          # https://serverfault.com/questions/453185/vagrant-virtualbox-dns-10-0-2-3-not-working
          # VBoxManage modifyvm "VM name" --natdnsproxy1 on
          # VBoxManage modifyvm "VM name" --natdnshostresolver1 on
          node.vm.provider :virtualbox do |vb, override|
            vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
            vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
            vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
          end
        end

        if !vars['default_nic_type'].nil?
          node.vm.provider :virtualbox do |vb|
            vb.default_nic_type = vars['default_nic_type']
          end
        end

        vars['synced_folders'].each do |synced_folder|
          # Share an additional folder to the guest VM. The first argument is
          # the path on the host to the actual folder. The second argument is
          # the path on the guest to mount the folder. And the optional third
          # argument is a set of non-required options.
          # node.vm.synced_folder "../data", "/vagrant_data"
          src = synced_folder["src"].to_s
          dest = synced_folder["dest"].to_s
          options = {}
          synced_folder.each do |key, value|
            if key != "src" && key != "dest"
              options[key.to_sym] = value
            end
          end

          puts "  Synced Folder: Source: #{src} -> Dest: #{dest}, Options: #{options}"
          node.vm.synced_folder src, dest, options
        end

        node.vm.provider :virtualbox do |vb|
          vb.gui = vars['gui']
          vb.memory = vars['memory']
          vb.cpus = vars['cpus']
        end

        node.vm.provider :libvirt do |libvirt|
          libvirt.memory = vars['memory']
          libvirt.cpus = vars['cpus']
        end

        vars['forwarded_ports'].each do |fwport|
          guest = fwport["guest"].to_i
          host = fwport["host"].to_i
          protocol = fwport.fetch("protocol", "tcp").to_s
          puts "  Forward Port: Host #{host} -> Guest #{guest}"
          node.vm.network :forwarded_port, guest: guest, host: host, protocol: protocol
        end

        vars['forwarded_port_range'].each do |fwportrange|
          from = fwportrange["from"].to_i
          to = fwportrange["to"].to_i
          protocol = fwportrange.fetch("protocol", "tcp").to_s
          puts "  Forward Port Range: #{from} .. #{to}, Protocol: #{protocol}"

          for j in from..to
            node.vm.network :forwarded_port, guest: j, host: j, protocol: protocol
          end
        end

        node.vm.network :private_network, ip: vm_private_ip #, nic_type: "Am79C973" # "virtio"

      end
    end
  end
end
