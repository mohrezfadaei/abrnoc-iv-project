Vagrant.configure("2") do |config|
  config.vm.define "debian12-test" do |node|
    node.vm.box = "debian/bookworm64"
    node.vm.hostname = "test-host"
    node.vm.network "private_network", ip: "192.168.56.10"

    node.vm.provider "libvirt" do |v|
      v.memory = 8192
      v.cpus = 4
      v.nested = true
      v.cpu_mode = "host-passthrough"

      v.storage :file, size: "10G", device: "vdb"
      v.storage :file, size: "10G", device: "vdc"
    end

    node.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/site.yml"
      ansible.become = true
    end
  end
end
