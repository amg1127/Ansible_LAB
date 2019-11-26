##################################################################
# Virtual machine configuration
vm_name_prefix = "vagrant_"
centos_version = "7"

##################################################################
# NetworkManager connection names, as presented by "nmcli" after virtual machine deployment
nmconn_1 = "enp0s3"
nmconn_2 = "'System enp0s8'"
nmconn_3 = "'System enp0s9'"
nmconn_4 = "'System enp0s10'"

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
  # Unofficial CentOS box
  # I am not using the official CentOS box because it has "net.ifnames=0" set in kernel command line.
  #config.vm.box = "centos/" + centos_version
  config.vm.box = "kane_project/centos" + centos_version
  
  # Disable shared folders feature
  config.vm.synced_folder ".", "/vagrant", disabled: true
  
  config.vm.define "gateway" do |guestVM|
    guestVM.vm.provider :virtualbox do |v|
      v.gui = true
      v.name = vm_name_prefix + "gateway" + "." + example_dns_name
      v.linked_clone = true
      v.customize ["modifyvm", :id, "--cpus", "2", "--memory", "1024", "--vram", "16"]
      v.customize ["storageattach", :id, "--storagectl", "IDE Controller", "--port", "1", "--device", "1", "--type", "dvddrive", "--medium", ENV['HOME'] + "/Downloads/CentOS-" + centos_version + "-x86_64.iso"]
    end
    guestVM.vm.hostname = "gateway" + "." + example_dns_name
    
    # Disable "UseDNS" in SSH server config - DNS server will be ready after running the Ansible playbook only...
    guestVM.vm.provision "shell", inline: "sed -i'' 's/^[[:space:]#]*UseDNS[[:space:]].*$/UseDNS no/' /etc/ssh/sshd_config ; systemctl reload-or-try-restart sshd"

    # I could not find instructions for setting both IPv4+IPv6 addresses on a single network interface.
    # I am going to assign IPv6 addresses manually instead...
    guestVM.vm.network "private_network", ip: gateway_net0_addr4 , netmask: "255.255.255.0", virtualbox__intnet: "net_" + vm_name_prefix + "0"
    guestVM.vm.network "private_network", ip: gateway_net32_addr4, netmask: "255.255.255.0", virtualbox__intnet: "net_" + vm_name_prefix + "32"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv6.method manual ipv6.address " + ip4to6(gateway_net0_addr4)  + "/64"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_3 + " ipv6.method manual ipv6.address " + ip4to6(gateway_net32_addr4) + "/64"
    
    # Avoid using NAT network builtin DNS server. I want to use the 'servera' VM instead...
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_1 + " ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes"
    guestVM.vm.provision "shell", inline: "nmcli con mod " + nmconn_2 + " ipv4.dns " + servera_net0_addr4 + " ipv6.dns " + ip4to6(servera_net0_addr4)
    
    # Commit interface configuration changes
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_1 + " && nmcli con up " + nmconn_1
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_2 + " && nmcli con up " + nmconn_2
    guestVM.vm.provision "shell", inline: "nmcli con down " + nmconn_3 + " && nmcli con up " + nmconn_3
  end

  config.vm.define "servera" do |guestVM|
    guestVM.vm.provider :virtualbox do |v|
      v.gui = true
      v.name = vm_name_prefix + "servera" + "." + example_dns_name
      v.linked_clone = true
      v.customize ["modifyvm", :id, "--cpus", "2", "--memory", "1024", "--vram", "16"]
      v.customize ["storageattach", :id, "--storagectl", "IDE Controller", "--port", "1", "--device", "1", "--type", "dvddrive", "--medium", ENV['HOME'] + "/Downloads/CentOS-" + centos_version + "-x86_64.iso"]
    end
    guestVM.vm.hostname = "servera" + "." + example_dns_name
    
    # Disable "UseDNS" in SSH server config - DNS server will be ready after running the Ansible playbook only...
    guestVM.vm.provision "shell", inline: "sed -i'' 's/^[[:space:]#]*UseDNS[[:space:]].*$/UseDNS no/' /etc/ssh/sshd_config ; systemctl reload-or-try-restart sshd"

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
      v.gui = true
      v.name = vm_name_prefix + "serverb" + "." + example_dns_name
      v.linked_clone = true
      v.customize ["modifyvm", :id, "--cpus", "2", "--memory", "1024", "--vram", "16"]
      v.customize ["storageattach", :id, "--storagectl", "IDE Controller", "--port", "1", "--device", "1", "--type", "dvddrive", "--medium", ENV['HOME'] + "/Downloads/CentOS-" + centos_version + "-x86_64.iso"]
    end
    guestVM.vm.hostname = "serverb" + "." + example_dns_name

    # Disable "UseDNS" in SSH server config - DNS server will be ready after running the Ansible playbook only...
    guestVM.vm.provision "shell", inline: "sed -i'' 's/^[[:space:]#]*UseDNS[[:space:]].*$/UseDNS no/' /etc/ssh/sshd_config ; systemctl reload-or-try-restart sshd"

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
      v.gui = true
      v.name = vm_name_prefix + "serverc" + "." + example_dns_name
      v.linked_clone = true
      v.customize ["modifyvm", :id, "--cpus", "2", "--memory", "1024", "--vram", "16"]
      v.customize ["storageattach", :id, "--storagectl", "IDE Controller", "--port", "1", "--device", "1", "--type", "dvddrive", "--medium", ENV['HOME'] + "/Downloads/CentOS-" + centos_version + "-x86_64.iso"]
    end
    guestVM.vm.hostname = "serverc" + "." + example_dns_name

    # Disable "UseDNS" in SSH server config - DNS server will be ready after running the Ansible playbook only...
    guestVM.vm.provision "shell", inline: "sed -i'' 's/^[[:space:]#]*UseDNS[[:space:]].*$/UseDNS no/' /etc/ssh/sshd_config ; systemctl reload-or-try-restart sshd"

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
      v.gui = true
      v.name = vm_name_prefix + "workstation" + "." + example_dns_name
      v.linked_clone = true
      v.customize ["modifyvm", :id, "--cpus", "2", "--memory", "2048", "--vram", "16"]
      v.customize ["storageattach", :id, "--storagectl", "IDE Controller", "--port", "1", "--device", "1", "--type", "dvddrive", "--medium", ENV['HOME'] + "/Downloads/CentOS-" + centos_version + "-x86_64.iso"]
    end
    guestVM.vm.hostname = "workstation" + "." + example_dns_name

    # Disable "UseDNS" in SSH server config - DNS server will be ready after running the Ansible playbook only...
    guestVM.vm.provision "shell", inline: "sed -i'' 's/^[[:space:]#]*UseDNS[[:space:]].*$/UseDNS no/' /etc/ssh/sshd_config ; systemctl reload-or-try-restart sshd"

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
  end
end

