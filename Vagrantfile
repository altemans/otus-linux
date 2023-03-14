# -*- mode: ruby -*-
# vim: set ft=ruby :

require 'open3'
require 'fileutils'

def get_vm_name(id)
        out, err = Open3.capture2e('VBoxManage list vms')
        raise out unless err.exitstatus.zero?
      
        path = path = File.dirname(__FILE__).split('/').last
        name = out.split(/\n/)
                  .select { |x| x.start_with? "\"#{path}_#{id}" }
                  .map { |x| x.tr('"', '') }
                  .map { |x| x.split(' ')[0].strip }
                  .first
      
        name
end

def controller_exists(name, controller_name)
        return false if name.nil?
      
        out, err = Open3.capture2e("VBoxManage showvminfo #{name}")
        raise out unless err.exitstatus.zero?
        out.split(/\n/)
           .select { |x| x.start_with? 'Storage Controller Name' }
           .map { |x| x.split(':')[1].strip }
           .any? { |x| x == controller_name }
end

def create_disks(vbox, name)
        # TODO: check that VM is created first time to avoid double vagrant up
       unless controller_exists(name, 'SATA Controller')
         vbox.customize ['storagectl', :id,
                         '--name', 'SATA Controller',
                         '--add', 'sata']
       end
      
        dir = "./"
        FileUtils.mkdir_p dir unless File.directory?(dir)
      
        disks = (1..5).map { |x| ["disk#{x}", '250'] }
      
        disks.each_with_index do |(name, size), i|
          file_to_disk = "#{dir}/#{name}.vdi"
          port = (i + 1).to_s
      
          unless File.exist?(file_to_disk)
            vbox.customize ['createmedium',
                            'disk',
                            '--filename',
                            file_to_disk,
                            '--size',
                            size,
                            '--format',
                            'VDI',
                            '--variant',
                            'standard']
          end
      
          vbox.customize ['storageattach', :id,
                          '--storagectl', 'SATA Controller',
                          '--port', port,
                          '--type', 'hdd',
                          '--medium', file_to_disk,
                          '--device', '0']
      
          vbox.customize ['setextradata', :id,
                          "VBoxInternal/Devices/ahci/0/Config/Port#{port}/SerialNumber",
                          name.ljust(20, '0')]
        end
      end

MACHINES = {
  :otuslinux => {
        :box_name => "centos/7",
        :ip_addr => '192.168.56.101'
  }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset
      box.vm.network "private_network", ip: boxconfig[:ip_addr]
      box.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "1024"]
        name = get_vm_name('server')
        create_disks(vb, name)
      end
      box.vm.provision "shell", inline: <<-SHELL
	mkdir -p ~root/.ssh
        cp ~vagrant/.ssh/auth* ~root/.ssh
	yum install -y mdadm smartmontools hdparm gdisk
        SHELL
      box.vm.provision "shell", path: "script.sh"
      end
  end
end

