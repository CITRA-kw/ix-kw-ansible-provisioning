# Created by Topology-Converter v4.6.8
#    Template Revision: v4.6.8
#    https://github.com/cumulusnetworks/topology_converter
#    using topology data from: ./topology.dot
#    built with the following args: ./topology_converter.py ./topology.dot
#
#    NOTE: in order to use this Vagrantfile you will need:
#       -Vagrant(v2.0.2+) installed: http://www.vagrantup.com/downloads
#       -the "helper_scripts" directory that comes packaged with topology-converter.py
#       -Virtualbox installed: https://www.virtualbox.org/wiki/Downloads



Vagrant.require_version ">= 2.0.2"

# Fix for Older versions of Vagrant to Grab Images from the Correct Location
unless Vagrant::DEFAULT_SERVER_URL.frozen?
  Vagrant::DEFAULT_SERVER_URL.replace('https://vagrantcloud.com')
end

$script = <<-SCRIPT
if grep -q -i 'cumulus' /etc/lsb-release &> /dev/null; then
    echo "### RUNNING CUMULUS EXTRA CONFIG ###"
    source /etc/lsb-release
    if [ -z /etc/app-release ]; then
        echo "  INFO: Detected NetQ TS Server"
        source /etc/app-release
        echo "  INFO: Running NetQ TS Appliance Version $APPLIANCE_VERSION"
    else
        if [[ $DISTRIB_RELEASE =~ ^2.* ]]; then
            echo "  INFO: Detected a 2.5.x Based Release"

            echo "  adding fake cl-acltool..."
            echo -e "#!/bin/bash\nexit 0" > /usr/bin/cl-acltool
            chmod 755 /usr/bin/cl-acltool

            echo "  adding fake cl-license..."
            echo -e "#!/bin/bash\nexit 0" > /usr/bin/cl-license
            chmod 755 /usr/bin/cl-license

            echo "  Disabling default remap on Cumulus VX..."
            mv -v /etc/init.d/rename_eth_swp /etc/init.d/rename_eth_swp.backup

            echo "### Rebooting to Apply Remap..."

        elif [[ $DISTRIB_RELEASE =~ ^3.* ]]; then
            echo "  INFO: Detected a 3.x Based Release"
            echo "### Disabling default remap on Cumulus VX..."
            mv -v /etc/hw_init.d/S10rename_eth_swp.sh /etc/S10rename_eth_swp.sh.backup &> /dev/null
            echo "### Disabling ZTP service..."
            systemctl stop ztp.service
            ztp -d 2>&1
            echo "### Resetting ZTP to work next boot..."
            ztp -R 2>&1
            echo "  INFO: Detected Cumulus Linux v$DISTRIB_RELEASE Release"
            if [[ $DISTRIB_RELEASE =~ ^3.[1-9].* ]]; then
                echo "### Fixing ONIE DHCP to avoid Vagrant Interface ###"
                echo "     Note: Installing from ONIE will undo these changes." 
                mkdir /tmp/foo
                mount LABEL=ONIE-BOOT /tmp/foo
                sed -i 's/eth0/eth1/g' /tmp/foo/grub/grub.cfg
                sed -i 's/eth0/eth1/g' /tmp/foo/onie/grub/grub-extra.cfg
                umount /tmp/foo
            fi
            if [[ $DISTRIB_RELEASE =~ ^3.[2-9].* ]]; then
                if [[ $(grep "vagrant" /etc/netd.conf | wc -l ) == 0 ]]; then
                    echo "### Giving Vagrant User Ability to Run NCLU Commands ###"
                    sed -i 's/users_with_edit = root, cumulus/users_with_edit = root, cumulus, vagrant/g' /etc/netd.conf
                    sed -i 's/users_with_show = root, cumulus/users_with_show = root, cumulus, vagrant/g' /etc/netd.conf
                fi
            fi
        fi
    fi
fi
echo "### DONE ###"
echo "### Rebooting Device to Apply Remap..."
nohup bash -c 'sleep 10; shutdown now -r "Rebooting to Remap Interfaces"' &
SCRIPT

Vagrant.configure("2") do |config|

  simid = 1522853971

  config.vm.provider "virtualbox" do |v|
    v.gui=false

  end




  ##### DEFINE VM for oob-mgmt-server #####
  config.vm.define "oob-mgmt-server" do |device|
    
    device.vm.hostname = "oob-mgmt-server" 
    
    device.vm.box = "CumulusCommunity/vx_oob_server"
    device.vm.box_version = "1.0.4"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_oob-mgmt-server"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 1024
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant"



    # NETWORK INTERFACES
      # link for eth1 --> oob-mgmt-switch:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net54", auto_config: false , :mac => "443839000057"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    
    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_oob_server.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:57 --> eth1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:57", NAME="eth1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for oob-mgmt-switch #####
  config.vm.define "oob-mgmt-switch" do |device|
    
    device.vm.hostname = "oob-mgmt-switch" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.5.3"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_oob-mgmt-switch"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for swp1 --> oob-mgmt-server:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net54", auto_config: false , :mac => "a00000000061"
      
      # link for swp2 --> server01:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net42", auto_config: false , :mac => "443839000043"
      
      # link for swp3 --> server02:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net47", auto_config: false , :mac => "44383900004c"
      
      
      # link for swp6 --> tec-ixp-leaf-01:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net20", auto_config: false , :mac => "443839000020"
      
      # link for swp7 --> tec-ixp-leaf-02:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net38", auto_config: false , :mac => "44383900003d"
      
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc13', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc14', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc15', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc16', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    
    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_oob_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:61 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:61", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:43 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:43", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:4c --> swp3"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:4c", NAME="swp3", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:20 --> swp6"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:20", NAME="swp6", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3d --> swp7"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3d", NAME="swp7", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end



  ##### DEFINE VM for tec-ixp-leaf-02 #####
  config.vm.define "tec-ixp-leaf-02" do |device|
    
    device.vm.hostname = "tec-ixp-leaf-02" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.5.3"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_tec-ixp-leaf-02"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp7
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net38", auto_config: false , :mac => "a00000000012"
      
      # link for swp1 --> server01:eth2
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net13", auto_config: false , :mac => "443839000015"
      
      # link for swp2 --> server02:eth2
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net15", auto_config: false , :mac => "443839000018"
      
      
      # link for swp53 --> tec-ixp-leaf-01:swp53
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net10", auto_config: false , :mac => "443839000011"
      
      # link for swp54 --> tec-ixp-leaf-01:swp54
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net1", auto_config: false , :mac => "443839000002"

      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    
    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:12 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:12", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:15 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:15", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:18 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:18", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:11 --> swp53"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:11", NAME="swp53", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:02 --> swp54"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:02", NAME="swp54", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end


  ##### DEFINE VM for tec-ixp-leaf-01 #####
  config.vm.define "tec-ixp-leaf-01" do |device|
    
    device.vm.hostname = "tec-ixp-leaf-01" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.5.3"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_tec-ixp-leaf-01"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp6
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net20", auto_config: false , :mac => "a00000000011"
      
      # link for swp1 --> server01:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net2", auto_config: false , :mac => "443839000003"
      
      # link for swp2 --> server02:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net12", auto_config: false , :mac => "443839000014"
            
      # link for swp53 --> tec-ixp-leaf-02:swp53
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net10", auto_config: false , :mac => "443839000010"
      
      # link for swp54 --> tec-ixp-leaf-02:swp54
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net1", auto_config: false , :mac => "443839000001"
      
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    
    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:11 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:11", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:03 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:03", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:14 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:14", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:10 --> swp53"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:10", NAME="swp53", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:01 --> swp54"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:01", NAME="swp54", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end



  ##### DEFINE VM for server01 #####
  config.vm.define "server01" do |device|
    
    device.vm.hostname = "server01" 
    
    device.vm.box = "yk0/ubuntu-xenial"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_server01"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net42", auto_config: false , :mac => "a00000000031"
      
      # link for eth1 --> tec-ixp-leaf-01:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net2", auto_config: false , :mac => "000300111101"
      
      # link for eth2 --> tec-ixp-leaf-02:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net13", auto_config: false , :mac => "000300111102"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    # Shorten Boot Process - Applies to Ubuntu Only - remove \"Wait for Network\"
    device.vm.provision :shell , inline: "sed -i 's/sleep [0-9]*/sleep 1/' /etc/init/failsafe.conf 2>/dev/null || true"

    
    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_server.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:31 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:31", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 00:03:00:11:11:01 --> eth1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="00:03:00:11:11:01", NAME="eth1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 00:03:00:11:11:02 --> eth2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="00:03:00:11:11:02", NAME="eth2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end



  ##### DEFINE VM for server02 #####
  config.vm.define "server02" do |device|
    
    device.vm.hostname = "server02" 
    
    device.vm.box = "yk0/ubuntu-xenial"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_server02"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp3
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net47", auto_config: false , :mac => "a00000000032"
      
      # link for eth1 --> tec-ixp-leaf-01:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net12", auto_config: false , :mac => "000300222201"
      
      # link for eth2 --> tec-ixp-leaf-02:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net15", auto_config: false , :mac => "000300222202"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    # Shorten Boot Process - Applies to Ubuntu Only - remove \"Wait for Network\"
    device.vm.provision :shell , inline: "sed -i 's/sleep [0-9]*/sleep 1/' /etc/init/failsafe.conf 2>/dev/null || true"

    
    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_server.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:32 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:32", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 00:03:00:22:22:01 --> eth1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="00:03:00:22:22:01", NAME="eth1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 00:03:00:22:22:02 --> eth2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="00:03:00:22:22:02", NAME="eth2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end



end
