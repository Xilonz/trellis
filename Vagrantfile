ANSIBLE_PATH = __dir__ # absolute path to Ansible directory on host machine
ANSIBLE_PATH_ON_VM = '/home/vagrant/trellis'.freeze # absolute path to Ansible directory on virtual machine

require File.join(ANSIBLE_PATH, 'lib', 'trellis', 'vagrant')
require File.join(ANSIBLE_PATH, 'lib', 'trellis', 'config')
require 'yaml'

vconfig = YAML.load_file("#{ANSIBLE_PATH}/vagrant.default.yml")

if File.exist?("#{ANSIBLE_PATH}/vagrant.local.yml")
  local_config = YAML.load_file("#{ANSIBLE_PATH}/vagrant.local.yml")
  vconfig.merge!(local_config) if local_config
end

ensure_plugins(vconfig.fetch('vagrant_plugins')) if vconfig.fetch('vagrant_install_plugins')

vms = YAML.load_file(File.join(File.dirname(__FILE__), 'vagrant.vms.yml'))

Vagrant.require_version '>= 2.1.0'

Vagrant.configure('2') do |config|

  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true

  # Required for NFS adn Hyper V to work
  if vconfig.fetch('vagrant_ip') == 'dhcp'
    cached_addresses = {}
    config.hostmanager.ip_resolver = proc do |vm, _resolving_vm|
      if cached_addresses[vm.name].nil?
        if vm.communicate.ready?
          cached_addresses[vm.name] = vm.provider.driver.read_guest_ip()['ip']
        end
      end
      cached_addresses[vm.name]
    end
  end

  vms.each do |vm_config|
    trellis_config = Trellis::Config.new(root_path: ANSIBLE_PATH, server: vm_config.fetch('name'))
    
    config.vm.define vm_config.fetch('name') do |srv|

      vconfig.merge!(vm_config) if vm_config

      srv.vm.box = vconfig.fetch('vagrant_box')
      srv.vm.box_version = vconfig.fetch('vagrant_box_version')
      srv.ssh.forward_agent = true
      srv.vm.post_up_message = post_up_message

      # Fix for: "stdin: is not a tty"
      # https://github.com/mitchellh/vagrant/issues/1673#issuecomment-28288042
      srv.ssh.shell = %(bash -c 'BASH_ENV=/etc/profile exec bash')

      # Required for NFS adn Hyper V to work
      if vconfig.fetch('vagrant_ip') == 'dhcp'
        srv.vm.network :private_network, type: 'dhcp', hostsupdater: 'skip'

        cached_addresses = {}
        srv.hostmanager.ip_resolver = proc do |vm, _resolving_vm|
          if cached_addresses[vm.name].nil?
            if vm.communicate.ready?
              cached_addresses[vm.name] = vm.provider.driver.read_guest_ip()['ip']
            end
          end
          cached_addresses[vm.name]
        end
      else
        srv.vm.network :private_network, ip: vconfig.fetch('vagrant_ip'), hostsupdater: 'skip'
      end

      main_hostname, *hostnames = trellis_config.site_hosts_canonical
      srv.vm.hostname = main_hostname

      if Vagrant.has_plugin?('vagrant-hostmanager') && !trellis_config.multisite_subdomains?
        redirects = trellis_config.site_hosts_redirects

        srv.hostmanager.enabled = true
        srv.hostmanager.manage_host = true
        srv.hostmanager.aliases = hostnames + redirects

      elsif Vagrant.has_plugin?('landrush') && trellis_config.multisite_subdomains?
        srv.landrush.enabled = true
        srv.landrush.tld = srv.vm.hostname
        hostnames.each { |host| srv.landrush.host host, vconfig.fetch('vagrant_ip') }
      else
        fail_with_message "vagrant-hostmanager missing, please install the plugin with this command:\nvagrant plugin install vagrant-hostmanager\n\nOr install landrush for multisite subdomains:\nvagrant plugin install landrush"
      end

      bin_path = File.join(ANSIBLE_PATH_ON_VM, 'bin')

      vagrant_mount_type = vconfig.fetch('vagrant_mount_type')

      extra_options = if vagrant_mount_type == 'smb'
        {
          smb_username: vconfig.fetch('vagrant_smb_username', 'vagrant'),
          smb_password: vconfig.fetch('vagrant_smb_password', 'vagrant'),
        }
      else
        {}
      end

      if vagrant_mount_type != 'nfs' || Vagrant::Util::Platform.wsl? || (Vagrant::Util::Platform.windows? && !Vagrant.has_plugin?('vagrant-winnfsd'))
        vagrant_mount_type = nil if vagrant_mount_type == 'nfs'
        trellis_config.wordpress_sites.each_pair do |name, site|
          srv.vm.synced_folder local_site_path(site), remote_site_path(name, site), owner: 'vagrant', group: 'www-data', mount_options: mount_options(vagrant_mount_type, dmode: 776, fmode: 775), type: vagrant_mount_type, **extra_options
        end

        srv.vm.synced_folder ANSIBLE_PATH, ANSIBLE_PATH_ON_VM, mount_options: mount_options(vagrant_mount_type, dmode: 755, fmode: 644), type: vagrant_mount_type, **extra_options
        srv.vm.synced_folder File.join(ANSIBLE_PATH, 'bin'), bin_path, mount_options: mount_options(vagrant_mount_type, dmode: 755, fmode: 755), type: vagrant_mount_type, **extra_options
      elsif !Vagrant.has_plugin?('vagrant-bindfs')
        fail_with_message "vagrant-bindfs missing, please install the plugin with this command:\nvagrant plugin install vagrant-bindfs"
      else
        srv.winnfsd.host_ip

        trellis_config.wordpress_sites.each_pair do |name, site|
          srv.vm.synced_folder local_site_path(site), nfs_path(name), type: 'nfs'
          srv.bindfs.bind_folder nfs_path(name), remote_site_path(name, site), u: 'vagrant', g: 'www-data', o: 'nonempty'
        end

        srv.vm.synced_folder ANSIBLE_PATH, '/ansible-nfs', type: 'nfs'
        srv.bindfs.bind_folder '/ansible-nfs', ANSIBLE_PATH_ON_VM, o: 'nonempty', p: '0644,a+D'
        srv.bindfs.bind_folder bin_path, bin_path, perms: '0755'
      end

      vconfig.fetch('vagrant_synced_folders', []).each do |folder|
        options = {
          type: folder.fetch('type', 'nfs'),
          create: folder.fetch('create', false),
          mount_options: folder.fetch('mount_options', [])
        }

        destination_folder = folder.fetch('bindfs', true) ? nfs_path(folder['destination']) : folder['destination']

        srv.vm.synced_folder folder['local_path'], destination_folder, options

        if folder.fetch('bindfs', true)
          srv.bindfs.bind_folder destination_folder, folder['destination'], folder.fetch('bindfs_options', {})
        end
      end

      provisioner = local_provisioning? ? :ansible_local : :ansible
      provisioning_path = local_provisioning? ? ANSIBLE_PATH_ON_VM : ANSIBLE_PATH

      srv.vm.provision provisioner do |ansible|
        if local_provisioning?
          ansible.install_mode = 'pip'
          ansible.provisioning_path = provisioning_path
          ansible.version = vconfig.fetch('vagrant_ansible_version')
        end

        ansible.compatibility_mode = '2.0'
        ansible.playbook = File.join(provisioning_path, 'dev.yml')
        ansible.galaxy_role_file = File.join(provisioning_path, 'requirements.yml') unless vconfig.fetch('vagrant_skip_galaxy') || ENV['SKIP_GALAXY']
        ansible.galaxy_roles_path = File.join(provisioning_path, 'vendor/roles')
        ansible.galaxy_command = 'ansible-galaxy install --role-file=%{role_file} --roles-path=%{roles_path}'

        ansible.groups = {
          'web' => vconfig.fetch('name'),
          'development' => vconfig.fetch('name')
        }

        ansible.tags = ENV['ANSIBLE_TAGS']
        ansible.extra_vars = { 'vagrant_version' => Vagrant::VERSION, 'env' => vconfig.fetch('name') }

        if (vars = ENV['ANSIBLE_VARS'])
          extra_vars = Hash[vars.split(',').map { |pair| pair.split('=') }]
          ansible.extra_vars.merge!(extra_vars)
        end

        if !Vagrant::Util::Platform.windows?
          srv.trigger.after :up do |trigger|
            # Add Vagrant ssh-config to ~/.ssh/config
            trigger.info = "Adding vagrant ssh-config for #{main_hostname } to ~/.ssh/config"
            trigger.ruby do
              update_ssh_config(main_hostname)
            end
          end
        end
      end

      # VirtualBox settings
      srv.vm.provider 'virtualbox' do |vb|
        vb.name  = srv.vm.hostname
        vb.customize ['modifyvm', :id, '--cpus', vconfig.fetch('vagrant_cpus')]
        vb.customize ['modifyvm', :id, '--memory', vconfig.fetch('vagrant_memory')]
        vb.customize ['modifyvm', :id, '--ioapic', vconfig.fetch('vagrant_ioapic', 'on')]

        # Fix for slow external network connections
        vb.customize ['modifyvm', :id, '--natdnshostresolver1', vconfig.fetch('vagrant_natdnshostresolver', 'on')]
        vb.customize ['modifyvm', :id, '--natdnsproxy1', vconfig.fetch('vagrant_natdnsproxy', 'on')]
      end

      # VMware Workstation/Fusion settings
      %w(vmware_fusion vmware_workstation).each do |provider|
        srv.vm.provider provider do |vmw, _override|
          vmw.name = srv.vm.hostname
          vmw.vmx['numvcpus'] = vconfig.fetch('vagrant_cpus')
          vmw.vmx['memsize'] = vconfig.fetch('vagrant_memory')
        end
      end

      # Parallels settings
      srv.vm.provider 'parallels' do |prl, _override|
        prl.name = srv.vm.hostname
        prl.cpus = vconfig.fetch('vagrant_cpus')
        prl.memory = vconfig.fetch('vagrant_memory')
        prl.update_guest_tools = true
      end

      # Hyper-V settings
      srv.vm.provider 'hyperv' do |h|
        h.vmname = srv.vm.hostname
        h.cpus = vconfig.fetch('vagrant_cpus')
        h.memory = vconfig.fetch('vagrant_memory')
        h.enable_virtualization_extensions = true
        h.linked_clone = true
      end
    end
  end
end