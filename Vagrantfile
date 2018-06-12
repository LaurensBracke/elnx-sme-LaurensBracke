# -*- mode: ruby -*-
# vi: ft=ruby :

require 'rbconfig'
require 'yaml'

# Set your default base box here
DEFAULT_BASE_BOX = 'bento/centos-7.4'

VAGRANTFILE_API_VERSION = '2'
PROJECT_NAME = '/' + File.basename(Dir.getwd)

# When set to `true`, Ansible will be forced to be run locally on the VM
# instead of from the host machine (provided Ansible is installed).
FORCE_LOCAL_RUN = false

hosts = YAML.load_file('vagrant-hosts.yml')

# {{{ Helper functions

def provision_ansible(config)
  if run_locally?
    # Provisioning configuration for shell script.
    config.vm.provision 'shell' do |sh|
      sh.path = 'scripts/playbook-win.sh'
    end
  else
    # Provisioning configuration for Ansible (for Mac/Linux hosts).
    config.vm.provision 'ansible' do |ansible|
      ansible.playbook = 'ansible/site.yml'
      ansible.sudo = true
    end
  end
end

def run_locally?
  windows_host? || FORCE_LOCAL_RUN
end

def windows_host?
  RbConfig::CONFIG['host_os'] =~ /mswin|mingw|cygwin/
end

# Set options for the network interface configuration. All values are
# optional, and can include:
# - ip (default = DHCP)
# - netmask (default value = 255.255.255.0
# - mac
# - auto_config (if false, Vagrant will not configure this network interface
# - intnet (if true, an internal network adapter will be created instead of a
#   host-only adapter)
def network_options(host)
  options = {}

  if host.key?('ip')
    options[:ip] = host['ip']
    options[:netmask] = host['netmask'] ||= '255.255.255.0'
  else
    options[:type] = 'dhcp'
  end

  options[:mac] = host['mac'].gsub(/[-:]/, '') if host.key?('mac')
  options[:auto_config] = host['auto_config'] if host.key?('auto_config')
  options[:virtualbox__intnet] = true if host.key?('intnet') && host['intnet']
  options
end

def custom_synced_folders(vm, host)
  return unless host.key?('synced_folders')
  folders = host['synced_folders']

  folders.each do |folder|
    vm.synced_folder folder['src'], folder['dest'], folder['options']
  end
end

# }}}

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.ssh.insert_key = false
  # VyOS Router
  config.vm.define 'router' do |router|
    router.vm.box = 'bertvv/vyos116'
    router.vm.network :private_network,
      ip: '192.0.2.254',
      netmask: '255.255.255.0',
      auto_config: false
    router.vm.network :private_network,
      ip: '172.16.255.254',
      netmask: '255.255.0.0',
      auto_config: false
    router.ssh.insert_key = false

    router.vm.provision "shell" do |sh|
      sh.path = "scripts/router-config.sh"
    end
  end
  hosts.each do |host|
    config.vm.define host['name'] do |node|
      node.vm.box = host['box'] ||= DEFAULT_BASE_BOX
      node.vm.box_url = host['box_url'] if host.key? 'box_url'

      node.vm.hostname = host['name']
      node.vm.network :private_network, network_options(host)
      custom_synced_folders(node.vm, host)

      node.vm.provider :virtualbox do |vb|
        # WARNING: if the name of the current directory is the same as the
        # host name, this will fail.
        vb.customize ['modifyvm', :id, '--groups', PROJECT_NAME]
      end
    end
  end
  provision_ansible(config)
end
