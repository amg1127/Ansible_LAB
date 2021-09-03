##################################################################
# Virtual machine configuration
vm_name_prefix = "vagrant_"

centos_dist_version = "stream8"
centos_box_version = "20210210.0"
centos_box_ide_ctrl = "IDE"
centos_box_sata_ctrl = "SATA"

##################################################################
# NetworkManager connection names, as presented by "nmcli" after virtual machine deployment
nmconn_1 = "'System eth0'"
nmconn_2 = "'System eth1'"
nmconn_3 = "'System eth2'"
nmconn_4 = "'System eth3'"

##################################################################
# DNS and LDAP domain names
example_dns_name = "example.com"
example_ldap_name = "dc=example,dc=com"

##################################################################
# Network settings for each virtual machine
network_prefix = "172.24"

gateway_net0_addr4  = network_prefix + ".0.1"
gateway_net32_addr4 = network_prefix + ".32.1"

servera_net0_addr4 = network_prefix + ".0.2"

serverb_net32_addr4   = network_prefix + ".32.3"
serverb_net64_0_addr4 = network_prefix + ".64.1"
serverb_net64_4_addr4 = network_prefix + ".64.5"

serverc_net32_addr4   = network_prefix + ".32.4"
serverc_net64_0_addr4 = network_prefix + ".64.2"
serverc_net64_4_addr4 = network_prefix + ".64.6"

workstation_net32_addr4 = network_prefix + ".32.48"

##################################################################
# Now, the provisioning code...

def ip4to6 (ipv4addr)
  return "fc00:" + ipv4addr.sub(/^(\d+)\.(\d+)\.(\d+)\.(\d+)$/, "\\1:\\2:\\3::\\4")
end

Vagrant.configure("2") do |config|
  config.vm.box = "centos/" + centos_dist_version
  config.vm.box_version = centos_box_version

  # Disable shared folders feature
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Disable "UseDNS" in SSH server config files - DNS server will be ready after running the Ansible playbook only...
  config.vm.provision "shell", inline: "sed -i'' 's/^[[:space:]#]*UseDNS[[:space:]].*$/UseDNS no/' /etc/ssh/sshd_config ; systemctl reload-or-try-restart sshd"

  # Modify nmcli profiles
  config.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " connection.interface-name enp0s3 || true"
  config.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " connection.interface-name enp0s8 || true"
  config.vm.provision "shell", inline: "nmcli con mod " + nmconn_3 + " connection.interface-name enp0s9 || true"
  config.vm.provision "shell", inline: "nmcli con mod " + nmconn_4 + " connection.interface-name enp0s10 || true"

  # Remove 'net.ifnames=0' from kernel command line and reboot the server
  config.vm.provision "shell", inline: "sed -r -i'' 's/^([[:space:]#]*GRUB_CMDLINE_LINUX=[\"'\\'']?(.*[[:space:]])?)net\.ifnames=0(([[:space:]].*)?[\"'\\'']?[[:space:]]*)\$/\\1\\3/' /etc/default/grub && grub2-mkconfig -o /boot/grub2/grub.cfg", reboot: true

  # Common VirtualBox provider parameters: GUI enabled, linked clone, standard CPU and memory configuration and ISO disk attachment
  config.vm.provider :virtualbox do |v|
    v.gui = true
    v.linked_clone = true
    v.customize ["modifyvm", :id, "--cpus", "2", "--memory", "1024", "--vram", "16", "--boot1", "disk"]
    #v.customize ["storagectl", :id, "--name", centos_box_sata_ctrl, "--hostiocache", "on"]
    v.customize ["storagectl", :id, "--name", centos_box_ide_ctrl, "--hostiocache", "on"]
    v.customize ["storageattach", :id, "--storagectl", centos_box_ide_ctrl, "--port", "0", "--device", "1", "--type", "dvddrive", "--medium", ENV['HOME'] + "/Downloads/CentOS-" + centos_dist_version + "-x86_64.iso"]
  end

  config.vm.define "gateway" do |guestVM|
    guestVM.vm.provider :virtualbox do |v|
      v.name = vm_name_prefix + "gateway" + "." + example_dns_name
    end
    guestVM.vm.hostname = "gateway" + "." + example_dns_name

    # firewall-cmd with nftables backend is not working...
    guestVM.vm.box = "centos/7"
    guestVM.vm.box_version = "2004.01"
    guestVM.vm.provider :virtualbox do |v|
      v.customize ["storageattach", :id, "--storagectl", centos_box_ide_ctrl, "--port", "0", "--device", "1", "--type", "dvddrive", "--medium", ENV['HOME'] + "/Downloads/CentOS-7-x86_64.iso"]
    end

    # I could not find instructions for setting both IPv4+IPv6 addresses on a single network interface.
    # I am going to assign IPv6 addresses manually instead...
    guestVM.vm.network "private_network", ip: gateway_net0_addr4 , netmask: "255.255.255.0", virtualbox__intnet: "net_" + vm_name_prefix + "0"
    guestVM.vm.network "private_network", ip: gateway_net32_addr4, netmask: "255.255.255.0", virtualbox__intnet: "net_" + vm_name_prefix + "32"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv6.method manual ipv6.address " + ip4to6(gateway_net0_addr4)  + "/64"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_3 + " ipv6.method manual ipv6.address " + ip4to6(gateway_net32_addr4) + "/64"

    # Avoid using NAT network builtin DNS server. I want to use the 'servera' VM instead...
    #guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.dns " + servera_net0_addr4 + " ipv6.dns " + ip4to6(servera_net0_addr4)

    # Commit interface configuration changes
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_1 + " && nmcli con up " + nmconn_1
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_2 + " && nmcli con up " + nmconn_2
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_3 + " && nmcli con up " + nmconn_3
  end

  config.vm.define "servera" do |guestVM|
    guestVM.vm.provider :virtualbox do |v|
      v.name = vm_name_prefix + "servera" + "." + example_dns_name
    end
    guestVM.vm.hostname = "servera" + "." + example_dns_name

    # I should use 389-ds instead... However, I haven't started my self-learning session yet...
    guestVM.vm.box = "centos/7"
    guestVM.vm.box_version = "2004.01"
    guestVM.vm.provider :virtualbox do |v|
      v.customize ["storageattach", :id, "--storagectl", centos_box_ide_ctrl, "--port", "0", "--device", "1", "--type", "dvddrive", "--medium", ENV['HOME'] + "/Downloads/CentOS-7-x86_64.iso"]
    end

    # I could not find instructions for setting both IPv4+IPv6 addresses on a single network interface.
    # I am going to assign IPv6 addresses manually instead...
    guestVM.vm.network "private_network", ip: servera_net0_addr4, netmask: "255.255.255.0", virtualbox__intnet: "net_" + vm_name_prefix + "0"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv6.method manual ipv6.address " + ip4to6(servera_net0_addr4) + "/64"

    # Avoid using Vagrant NAT network interface for traffic going to the internet. I want to use the 'gateway' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-routes yes ipv6.ignore-auto-routes yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.gateway " + gateway_net0_addr4 + " ipv6.gateway " + ip4to6(gateway_net0_addr4)

    # Avoid using NAT network builtin DNS server. I want to use the 'servera' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.dns " + servera_net0_addr4 + " ipv6.dns " + ip4to6(servera_net0_addr4)

    # Commit interface configuration changes
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_1 + " && nmcli con up " + nmconn_1
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_2 + " && nmcli con up " + nmconn_2
  end

  config.vm.define "serverb" do |guestVM|
    guestVM.vm.provider :virtualbox do |v|
      v.name = vm_name_prefix + "serverb" + "." + example_dns_name
    end
    guestVM.vm.hostname = "serverb" + "." + example_dns_name

    # I could not find instructions for setting both IPv4+IPv6 addresses on a single network interface.
    # I am going to assign IPv6 addresses manually instead...
    guestVM.vm.network "private_network", ip: serverb_net32_addr4, netmask: "255.255.255.0", virtualbox__intnet: "net_" + vm_name_prefix + "32"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv6.method manual ipv6.address " + ip4to6(serverb_net32_addr4) + "/64"

    # Avoid using Vagrant NAT network interface for traffic going to the internet. I want to use the 'gateway' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-routes yes ipv6.ignore-auto-routes yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.gateway " + gateway_net32_addr4 + " ipv6.gateway " + ip4to6(gateway_net32_addr4)

    # Avoid using NAT network builtin DNS server. I want to use the 'servera' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.dns " + servera_net0_addr4 + " ipv6.dns " + ip4to6(servera_net0_addr4)

    # I am planning to setup link aggregation later...
    guestVM.vm.network "private_network", ip: serverb_net64_0_addr4, netmask: "255.255.255.252", virtualbox__intnet: "net_" + vm_name_prefix + "64", auto_config: false
    guestVM.vm.network "private_network", ip: serverb_net64_4_addr4, netmask: "255.255.255.252", virtualbox__intnet: "net_" + vm_name_prefix + "64", auto_config: false

    # Commit interface configuration changes
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_1 + " && nmcli con up " + nmconn_1
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_2 + " && nmcli con up " + nmconn_2
  end

  config.vm.define "serverc" do |guestVM|
    guestVM.vm.provider :virtualbox do |v|
      v.name = vm_name_prefix + "serverc" + "." + example_dns_name
    end
    guestVM.vm.hostname = "serverc" + "." + example_dns_name

    # I could not find instructions for setting both IPv4+IPv6 addresses on a single network interface.
    # I am going to assign IPv6 addresses manually instead...
    guestVM.vm.network "private_network", ip: serverc_net32_addr4, netmask: "255.255.255.0", virtualbox__intnet: "net_" + vm_name_prefix + "32"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv6.method manual ipv6.address " + ip4to6(serverc_net32_addr4) + "/64"

    # Avoid using Vagrant NAT network interface for traffic going to the internet. I want to use the 'gateway' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-routes yes ipv6.ignore-auto-routes yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.gateway " + gateway_net32_addr4 + " ipv6.gateway " + ip4to6(gateway_net32_addr4)

    # Avoid using NAT network builtin DNS server. I want to use the 'servera' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.dns " + servera_net0_addr4 + " ipv6.dns " + ip4to6(servera_net0_addr4)

    # I am planning to setup link aggregation later...
    guestVM.vm.network "private_network", ip: serverc_net64_0_addr4, netmask: "255.255.255.252", virtualbox__intnet: "net_" + vm_name_prefix + "64", auto_config: false
    guestVM.vm.network "private_network", ip: serverc_net64_4_addr4, netmask: "255.255.255.252", virtualbox__intnet: "net_" + vm_name_prefix + "64", auto_config: false

    # Commit interface configuration changes
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_1 + " && nmcli con up " + nmconn_1
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_2 + " && nmcli con up " + nmconn_2
  end

  config.vm.define "workstation" do |guestVM|
    guestVM.vm.provider :virtualbox do |v|
      v.name = vm_name_prefix + "workstation" + "." + example_dns_name
      # Workstation will have additional memory because I am going to install graphical environment
      v.customize ["modifyvm", :id, "--cpus", "2", "--memory", "2048", "--vram", "16"]
    end
    guestVM.vm.hostname = "workstation" + "." + example_dns_name

    # I could not find instructions for setting both IPv4+IPv6 addresses on a single network interface.
    # I am going to assign IPv6 addresses manually instead...
    guestVM.vm.network "private_network", ip: workstation_net32_addr4 , netmask: "255.255.255.0", virtualbox__intnet: "net_" + vm_name_prefix + "32"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv6.method manual ipv6.address " + ip4to6(workstation_net32_addr4) + "/64"

    # Avoid using Vagrant NAT network interface for traffic going to the internet. I want to use the 'gateway' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-routes yes ipv6.ignore-auto-routes yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.gateway " + gateway_net32_addr4 + " ipv6.gateway " + ip4to6(gateway_net32_addr4)

    # Avoid using NAT network builtin DNS server. I want to use the 'servera' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.dns " + servera_net0_addr4 + " ipv6.dns " + ip4to6(servera_net0_addr4)

    # Commit interface configuration changes
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_1 + " && nmcli con up " + nmconn_1
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_2 + " && nmcli con up " + nmconn_2

    # Because this is the last machine, launch the Ansible provisioner from it
    guestVM.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "./playbook.yml"
      ansible.limit = "all"
      ansible.extra_vars = {
        example_dns_name: example_dns_name,
        example_ldap_name: example_ldap_name,
        example_hosts: {
          gateway:     { addr4net0:  gateway_net0_addr4, addr4net32: gateway_net32_addr4, addr6net0: ip4to6(gateway_net0_addr4), addr6net32: ip4to6(gateway_net32_addr4) },
          servera:     { addr4net0:  servera_net0_addr4,      addr6net0:  ip4to6(servera_net0_addr4)      },
          serverb:     { addr4net32: serverb_net32_addr4,     addr6net32: ip4to6(serverb_net32_addr4)     },
          serverc:     { addr4net32: serverc_net32_addr4,     addr6net32: ip4to6(serverc_net32_addr4)     },
          workstation: { addr4net32: workstation_net32_addr4, addr6net32: ip4to6(workstation_net32_addr4) }
        }
      }
      ansible.groups = {
        "all:children" => [ "servers", "workstations" ],
        "servers" => [ "gateway", "servera", "serverb", "serverc" ],
        "workstations" => [ "workstation" ]
      }
      ansible.vault_password_file = "./vault_password_file"
    end
  end
end

