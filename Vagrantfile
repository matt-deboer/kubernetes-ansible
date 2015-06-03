# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
#
#
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# helper functions
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

# Deep convert keys to symbols for use in interpolation
def symbolize(h)
  case h
  when Hash
    Hash[
      h.map do |k, v|
        [ k.respond_to?(:to_sym) ? k.to_sym : k, symbolize(v) ]
      end
    ]
  when Enumerable
    h.map { |v| symbolize(v) }
  else
    h
  end
end
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# configs, constants
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
VAGRANTFILE_API_VERSION = "2"

base_dir = File.expand_path(File.dirname(__FILE__))
config = symbolize YAML.load_file('config.yml')

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# build ansible groups and cluster definition from configs
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
cluster= {}
ansible_groups = {
  "masters" => [],
  "minions" => [],
  "vagrant:children" => ["masters", "minions"]
}

(1..config[:masters][:count]).each do |i|
  hostname = "k8s-master-%02d" % i
  cluster[hostname] = { 
    :ip => "%{base_ip}#{10+i}" % config[:masters],
    :cpus => config[:masters][:cpus], 
    :mem => config[:masters][:mem],
    :box => config[:masters][:box]
  }
  ansible_groups["masters"].push hostname
end

(1..config[:minions][:count]).each do |i|
  hostname = "k8s-node-%02d" % i
  cluster[hostname] = { 
    :ip => "%{base_ip}#{100+i}" % config[:minions],
    :cpus => config[:minions][:cpus], 
    :mem => config[:minions][:mem],
    :box => config[:minions][:box]
  }
  ansible_groups["minions"].push 
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  cluster.each_with_index do |(hostname, info), index|

    config.ssh.insert_key = false
    config.ssh.forward_agent = true
    config.ssh.private_key_path = [
      '~/.vagrant.d/insecure_private_key', 
      '~/.ssh/id_rsa'
    ]

    config.vm.define hostname do |cfg|

      cfg.vm.provider :virtualbox do |vb, override|
        
        override.vm.box = info[:box][:name]
        if info[:box].has_key? :url
          override.vm.box_url = info[:box][:url]
        end
        override.vm.network :private_network, ip: "#{info[:ip]}"
        override.vm.hostname = hostname
        vb.name = 'vagrant-mesos-' + hostname
        vb.customize ["modifyvm", :id, "--memory", info[:mem], "--cpus", info[:cpus], "--hwvirtex", "on" ]
      end

      # provision nodes with ansible, only on the
      # last host to tigger parallel provision
      if index == cluster.size - 1
        cfg.vm.provision :ansible do |ansible|
          ansible.verbose = "vv"
          ansible.raw_arguments = ['--timeout=300']
          # ansible.raw_ssh_args = ['-o UserKnownHostsFile=/dev/null']
          #ansible.inventory_path = base_dir + "/inventory/vagrant"
          ansible.extra_vars = {
            authorized_key: Dir.home + '/.ssh/id_rsa.pub'
          }
          ansible.groups = ansible_groups
          ansible.playbook = base_dir + "/setup.yml"
          # ansible.sudo = true
          ansible.limit = 'all'
          # ansible.limit = "#{info[:ip]}" # Ansible hosts are identified by ip
        end
      end
    end
  end
end
