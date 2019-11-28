Vagrant.configure("2") do |config|
  config.ssh.insert_key = false

  config.vm.define "haproxy" do |haproxy|
    haproxy.vm.box = "ubuntu/bionic64"
    haproxy.vm.hostname = "haproxy"
    haproxy.vm.network "private_network", ip: "10.10.10.10", netmask: "255.255.255.0"

    # virtualbox setup
    haproxy.vm.provider "virtualbox" do |vb|
      vb.name = "haproxy"

      # Set CPU and Ram
      vb.customize ["modifyvm", :id, "--cpus", "1"]
      vb.customize ["modifyvm", :id, "--memory", "512"]
    end
  end

  config.vm.define "docker01" do |docker01|
    docker01.vm.box = "ubuntu/bionic64"
    docker01.vm.hostname = "docker01"
    docker01.vm.network "private_network", ip: "10.10.10.20", netmask: "255.255.255.0"
    docker01.disksize.size = '20GB'

    # virtualbox setup
    docker01.vm.provider "virtualbox" do |vb|
      vb.name = "docker01"

      # Set CPU and Ram
      vb.customize ["modifyvm", :id, "--cpus", "1"]
      vb.customize ["modifyvm", :id, "--memory", "1024"]

      # Get disk path
      line = `VBoxManage list systemproperties | grep "Default machine folder"`
      vb_machine_folder = line.split(':')[1].strip()
      second_disk = File.join(vb_machine_folder, vb.name, 'disk2.vdi')

      # Create disk
      unless File.exist?(second_disk)
        vb.customize ['createmedium', 'disk', '--filename', second_disk, '--size', 10 * 1024]
      end

      # Attach disk to VM
      vb.customize ['storageattach', :id, '--storagectl', 'SCSI', '--port', 2, '--type', 'hdd', '--medium', second_disk]
    end
  end

  config.vm.define "docker02" do |docker02|
    docker02.vm.box = "ubuntu/bionic64"
    docker02.vm.hostname = "docker02"
    docker02.vm.network "private_network", ip: "10.10.10.30", netmask: "255.255.255.0"
    docker02.disksize.size = '20GB'

    # virtualbox setup
    docker02.vm.provider "virtualbox" do |vb|
      vb.name = "docker02"

      # Set CPU and Ram
      vb.customize ["modifyvm", :id, "--cpus", "1"]
      vb.customize ["modifyvm", :id, "--memory", "1024"]

      # Get disk path
      line = `VBoxManage list systemproperties | grep "Default machine folder"`
      vb_machine_folder = line.split(':')[1].strip()
      second_disk = File.join(vb_machine_folder, vb.name, 'disk2.vdi')

      # Create disk
      unless File.exist?(second_disk)
        vb.customize ['createmedium', 'disk', '--filename', second_disk, '--size', 10 * 1024]
      end

      # Attach disk to VM
      vb.customize ['storageattach', :id, '--storagectl', 'SCSI', '--port', 2, '--type', 'hdd', '--medium', second_disk]
    end
  end

  config.vm.define "docker03" do |docker03|
    docker03.vm.box = "ubuntu/bionic64"
    docker03.vm.hostname = "docker03"
    docker03.vm.network "private_network", ip: "10.10.10.40", netmask: "255.255.255.0"
    docker03.disksize.size = '20GB'

    # virtualbox setup
    docker03.vm.provider "virtualbox" do |vb|
      vb.name = "docker03"

      # Set CPU and Ram
      vb.customize ["modifyvm", :id, "--cpus", "1"]
      vb.customize ["modifyvm", :id, "--memory", "1024"]

      # Get disk path
      line = `VBoxManage list systemproperties | grep "Default machine folder"`
      vb_machine_folder = line.split(':')[1].strip()
      second_disk = File.join(vb_machine_folder, vb.name, 'disk2.vdi')

      # Create disk
      unless File.exist?(second_disk)
        vb.customize ['createmedium', 'disk', '--filename', second_disk, '--size', 10 * 1024]
      end

      # Attach disk to VM
      vb.customize ['storageattach', :id, '--storagectl', 'SCSI', '--port', 2, '--type', 'hdd', '--medium', second_disk]
    end
  end
end
