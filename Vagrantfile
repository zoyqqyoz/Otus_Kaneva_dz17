disk = './BackupDisk.vdi'

Vagrant.configure("2") do |config|

config.vm.synced_folder ".", "/vagrant", disabled: true

config.vm.define "backup" do |backup|

    backup.vm.box = "centos/7"

    backup.vm.network "private_network", ip: "192.168.56.20"

    backup.vm.hostname = "backup-server"

    backup.vm.provider :virtualbox do |vb|

      vb.customize ["modifyvm", :id, "--memory", "1024"]

      vb.customize ["modifyvm", :id, "--cpus", "2"]
	
	unless File.exist?(disk)
	  
	  vb.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', 2 * 1024]
		
	  end
	  
	  vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 1, '--device', 1, '--type', 'hdd', '--medium', disk]

      end

   backup.vm.provision "shell", inline: <<-SHELL

       mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh

       sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config

       systemctl restart sshd

       timedatectl set-timezone Europe/Moscow

       yum install epel-release -y

       yum install ntp -y && systemctl start ntpd && systemctl enable ntpd

       yum install borgbackup -y     
  
       useradd -m borg

       

    SHELL

    end

 

    config.vm.define "client" do |client|

      client.vm.box = "centos/7"

      client.vm.network "private_network", ip: "192.168.56.21"

      client.vm.hostname = "client"

      client.vm.provider :virtualbox do |vb|

      vb.customize ["modifyvm", :id, "--memory", "512"]

      vb.customize ["modifyvm", :id, "--cpus", "2"]

      end


      client.vm.provision "shell", inline: <<-SHELL

       mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh

       sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config

       systemctl restart sshd

       timedatectl set-timezone Europe/Moscow

       yum install epel-release -y
       
       yum install ntp -y && systemctl start ntpd && systemctl enable ntpd

       yum install borgbackup -y

       useradd -m borg

            
    SHELL

  end
end
