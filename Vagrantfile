Vagrant.configure("2") do |config|
    (1..1).each do |i|
        config.vm.define "master-#{i}" do |master|
            master.vm.box = "ubuntu/jammy64"
            master.vm.hostname = "master-#{i}"
            master.vm.network "private_network", ip: "172.89.0.1#{i}"

            master.ssh.insert_key = false
            master.ssh.private_key_path = ['~/.vagrant.d/insecure_private_key', '~/.ssh/id_rsa']
            master.vm.provision "file", source: "~/.ssh/id_rsa.pub", destination: "~/.ssh/authorized_keys"

            master.vm.provider :libvirt do |libvirt|
                libvirt.driver = "kvm"
                libvirt.memory = 2048
                libvirt.cpus = 2

                libvirt.disk_bus = "virtio"
                libvirt.nic_model_type = "virtio"
                libvirt.random_model = "virtio"
            end
        end
    end

    (1..2).each do |i|
        config.vm.define "worker-#{i}" do |worker|
            worker.vm.box = "ubuntu/jammy64"
            worker.vm.hostname = "worker-#{i}"
            worker.vm.network "private_network", ip: "172.89.0.2#{i}"

            worker.ssh.insert_key = false
            worker.ssh.private_key_path = ['~/.vagrant.d/insecure_private_key', '~/.ssh/id_rsa']
            worker.vm.provision "file", source: "~/.ssh/id_rsa.pub", destination: "~/.ssh/authorized_keys"

            master.vm.provider :libvirt do |libvirt|
                libvirt.driver = "kvm"
                libvirt.memory = 2048
                libvirt.cpus = 2

                libvirt.disk_bus = "virtio"
                libvirt.nic_model_type = "virtio"
                libvirt.random_model = "virtio"
            end
        end
    end
end
