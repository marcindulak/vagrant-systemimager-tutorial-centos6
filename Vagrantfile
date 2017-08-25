# -*- mode: ruby -*-
# vi: set ft=ruby :

OSCAR_RELEASE="6.1.2r11073-1"

goldenclientIP="192.168.10.10"
serverIP="192.168.10.5"
nodes=[["node101", "192.168.10.101", "080027000101"],
       ["node102", "192.168.10.102", "080027000102"],
       ["node103", "192.168.10.103", "080027000103"]]

# http://stackoverflow.com/questions/23926945/specify-headless-or-gui-from-command-line
def gui_enabled?
  !ENV.fetch('GUI', '').empty?
end

Vagrant.configure(2) do |config|
  # goldenclient
  config.vm.define "goldenclient" do |goldenclient|
    if !ENV.fetch('CLIENT_NO_LVM', '').empty?
      # lvmcreate hangs during systeminamger installation of a centos6 node - we use a box without lvm
      goldenclient.vm.box = "brianbirkinbine/vagrant-centos-65-x86_64-minimal"
      goldenclient.vm.box_url = "http://files.brianbirkinbine.com/vagrant-centos-65-x86_64-minimal.box"
    else
      goldenclient.vm.box = "puppetlabs/centos-6.6-64-nocm"
      goldenclient.vm.box_url = "puppetlabs/centos-6.6-64-nocm"
    end
    goldenclient.vm.network "private_network", ip: goldenclientIP, auto_config: false
    goldenclient.vm.provider "virtualbox" do |v|
      v.memory = 256
      v.cpus = 1
    end
  end
  # server
  config.vm.define "server" do |server|
    server.vm.box = "puppetlabs/centos-6.6-64-nocm"
    server.vm.box_url = 'puppetlabs/centos-6.6-64-nocm'
    server.vm.network "private_network", ip: serverIP, auto_config: false
    server.vm.provider "virtualbox" do |v|
      v.memory = 128
      v.cpus = 1
    end
  end
  # nodes
  # Based on https://github.com/simonjohansson/pxe-coreos-vagrant under unknown license ...
  nodes.collect.each_with_index do |data, index|
    config.vm.define "#{data[0]}" do |node|
      ip = data[1]
      mac = data[2]

      # For boxes with metadata, you cannot override the name
      #node.vm.box = data[0]

      # Use the golden client image as a placeholder for a pxe booted node
      # One could also use "special" empty image like https://atlas.hashicorp.com/steigr/boxes/pxe
      # but then one must make sure enough storage is available to the node
      if !ENV.fetch('CLIENT_NO_LVM', '').empty?
        node.vm.box = "brianbirkinbine/vagrant-centos-65-x86_64-minimal"
        node.vm.box_url = "http://files.brianbirkinbine.com/vagrant-centos-65-x86_64-minimal.box"
      else
        node.vm.box = "puppetlabs/centos-6.6-64-nocm"
        node.vm.box_url = "puppetlabs/centos-6.6-64-nocm"
      end

      # https://docs.vagrantup.com/v2/virtualbox/boxes.html
      # second adapter used for pxe boot
      node.vm.network "private_network", :adapter => 2, ip: ip, :mac => mac, auto_config: false

      node.vm.synced_folder '.', '/vagrant', disabled: true

      node.vm.boot_timeout = 1800  # 300 seconds default; includes systemimager installation time

      node.vm.provider "virtualbox" do |vb, override|
        # put primary network interface into hostonly network segement
        vb.customize ["modifyvm", :id, "--nic2", "hostonly"]
        vb.customize ["modifyvm", :id, "--hostonlyadapter1", "vboxnet0"]
        # set pxe boot before disk
        vb.customize ["modifyvm", :id, "--boot2", "disk"]
        vb.customize ["modifyvm", :id, "--boot1", "net"]
        # use second interface for pxe boot
        vb.customize ["modifyvm", :id, "--nicbootprio1", "0"]
        vb.customize ["modifyvm", :id, "--nicbootprio2", "1"]
        # show boot menu
        #vb.customize ["modifyvm", :id, "--biosbootmenu", "messageandmenu"]
        #vb.customize ["modifyvm", :id, "--bioslogodisplaytime", "20000"]  # 20 sec
        # supervise imaging process in VBox's gui
        vb.gui = gui_enabled?
      end

      node.vm.provider "virtualbox" do |vb|
        vb.memory = 512  # No partition found (1) No filesystem could mount root
        vb.cpus = 1
      end

      # keep using the default insecure key
      node.ssh.insert_key = false

    end
  end
  # disable IPv6 on Linux
  $linux_disable_ipv6 = <<SCRIPT
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
SCRIPT
  # permanently disable IPv6 on Linux
  $linux_disable_ipv6_permanently = <<SCRIPT
echo "# Disable ipv6" >> /etc/sysctl.conf
echo net.ipv6.conf.default.disable_ipv6 = 1 >> /etc/sysctl.conf
echo net.ipv6.conf.all.disable_ipv6 = 1 >> /etc/sysctl.conf
echo net.ipv6.conf.lo.disable_ipv6 = 1 >> /etc/sysctl.conf
SCRIPT
  # setenforce 0
  $setenforce_0 = <<SCRIPT
if test `getenforce` = 'Enforcing'; then setenforce 0; fi
#sed -Ei 's/^SELINUX=.*/SELINUX=Permissive/' /etc/selinux/config
SCRIPT
  # stop iptables
  $service_iptables_stop = <<SCRIPT
service iptables stop
SCRIPT
  # configure the second vagrant eth interface
  $ifcfg_device = <<SCRIPT
IPADDR=$1
NETMASK=$2
DEVICE=$3
cat <<END >> /etc/sysconfig/network-scripts/ifcfg-$DEVICE
NM_CONTROLLED=yes
BOOTPROTO=none
ONBOOT=yes
IPADDR=$IPADDR
NETMASK=$NETMASK
DEVICE=$DEVICE
PEERDNS=no
END
ARPCHECK=no /sbin/ifup $DEVICE 2> /dev/null
SCRIPT
  # dhcp server
  $etc_sysconfig_dhcpd = <<SCRIPT
INTERFACE=$1
sed -Ei "s/^DHCPDARGS=.*/DHCPDARGS=${INTERFACE}/" /etc/sysconfig/dhcpd
SCRIPT
  # .ssh permission
  $chmod_600_dotssh = <<SCRIPT
sudo su -c "chmod -R 600 ~$1/.ssh/"
sudo su -c "chmod 700 ~$1/.ssh"
SCRIPT
  # .ssh ownership
  $chown_dotssh = <<SCRIPT
sudo su -c "chown -R $1:$2 ~$1/.ssh/"
SCRIPT
  # /etc/hosts
  $etc_hosts = <<SCRIPT
IP=$1
NAME=$2
cat <<END >> /etc/hosts
$IP $NAME
END
SCRIPT
  # EPEL
  $epel6 = <<SCRIPT
yum clean all
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm
SCRIPT
  # oscar-relase
  $oscar_release_rhel6 = <<SCRIPT
OSCAR_RELEASE=$1
yum clean all
yum -y install http://svn.oscar.openclustergroup.org/repos/unstable/rhel-6-x86_64/oscar-release-${OSCAR_RELEASE}.el6.noarch.rpm
SCRIPT
  $etc_dhcp_dhcpd_conf = <<SCRIPT
SERVER=$1
NETWORK=$2
NETMASK=$3
BROADCAST=$4
FIRSTCLIENT=$5
LASTCLIENT=$6
cat <<END >> /etc/dhcp/dhcpd.conf
# general options
authoritative;
ddns-update-style none;

# Imageserver
option option-140 code 140 = ip-address;

# option-140 is the IP address of your SystemImager image server
option option-140 ${SERVER};

# next-server is your network boot server
next-server ${SERVER};

# set lease time to infinite (-1)
default-lease-time -1;

filename "pxelinux.0";

ddns-update-style none;

subnet ${NETWORK} netmask ${NETMASK} {
  option broadcast-address ${BROADCAST};
  option routers ${SERVER};
  option domain-name "localdomain";
  pool {
    range ${FIRSTCLIENT} ${LASTCLIENT};
  }
}

host node101 {
   option host-name "node101";
   hardware ethernet 08:00:27:00:01:01;
   fixed-address 192.168.10.101;
}
host node102 {
   option host-name "node102";
   hardware ethernet 08:00:27:00:01:02;
   fixed-address 192.168.10.102;
}
host node103 {
   option host-name "node103";
   hardware ethernet 08:00:27:00:01:03;
   fixed-address 192.168.10.103;
}
END
SCRIPT
  $tftpboot_pxelinux_cfg_default = <<SCRIPT
cat <<'END' >> /var/lib/tftpboot/pxelinux.cfg/default.node.install
DEFAULT systemimager
LABEL systemimager
DISPLAY message.txt
PROMPT 1
TIMEOUT 50
KERNEL kernel.node
APPEND vga=extended initrd=initrd.img.node root=/dev/ram ramdisk_size=65536 tmpfs_size=2048M ramdisk_blocksize=1024
END
cd /var/lib/tftpboot/pxelinux.cfg
ln -s default.node.install default
cp -p /etc/systemimager/pxelinux.cfg/message.txt /var/lib/tftpboot
SCRIPT
  config.vm.define "goldenclient" do |goldenclient|
    goldenclient.vm.provision :shell, :inline => "hostname goldenclient", run: "always"
    goldenclient.vm.provision "shell" do |s|
      s.inline = $etc_hosts
      s.args   = [serverIP, "server"]
    end
    goldenclient.vm.provision "shell" do |s|
      s.inline = $etc_hosts
      s.args   = [goldenclientIP, "goldenclient"]
    end
    nodes.collect.each_with_index do |data, index|
      ip = data[1]
      name = data[0]
      goldenclient.vm.provision "shell" do |s|
        s.inline = $etc_hosts
        s.args   = [ip, name]
      end
    end
    goldenclient.vm.provision :shell, :inline => $linux_disable_ipv6,  run: "always"
    goldenclient.vm.provision :shell, :inline => $linux_disable_ipv6_permanently
    goldenclient.vm.provision :shell, :inline => $service_iptables_stop, run: "always"
    goldenclient.vm.provision :shell, :inline => $setenforce_0, run: "always"
    goldenclient.vm.provision "shell" do |s|
      s.inline = $ifcfg_device
      s.args   = [goldenclientIP, "255.255.255.0", "eth1"]
    end
    goldenclient.vm.provision :shell, :inline => "ifup eth1", run: "always"
    # restarting network fixes RTNETLINK answers: File exists
    goldenclient.vm.provision :shell, :inline => 'service network restart'
    goldenclient.vm.provision :file, source: "~/.vagrant.d/insecure_private_key", destination: "~vagrant/.ssh/id_rsa"
    goldenclient.vm.provision :shell, :inline => "cp -rp ~vagrant/.ssh ~root/"
    goldenclient.vm.provision :shell, :inline => "yum -y install wget"
    goldenclient.vm.provision :shell, :inline => "wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O ~root/.ssh/authorized_keys"
    goldenclient.vm.provision "shell" do |s|
      s.inline = $chmod_600_dotssh
      s.args   = ["root"]
    end
    goldenclient.vm.provision "shell" do |s|
      s.inline = $chown_dotssh
      s.args   = ["root", "root"]
    end
    goldenclient.vm.provision :shell, :inline => $epel6
    goldenclient.vm.provision "shell" do |s|
      s.inline = $oscar_release_rhel6
      s.args   = [OSCAR_RELEASE]
    end
    goldenclient.vm.provision :shell, :inline => "yum -y install systemimager-client systemimager-x86_64initrd_template"
    goldenclient.vm.provision :shell, :inline => "yum -y install yum-plugin-versionlock"
    goldenclient.vm.provision :shell, :inline => "yum versionlock systemconfigurator systemimager*"
    # sshpass needed for the first login after creating new authorized_keys
    goldenclient.vm.provision :shell, :inline => "yum -y install sshpass"
    # install additional software (mpi4py)
    goldenclient.vm.provision :shell, :inline => "yum -y install yum-utils"
    goldenclient.vm.provision :shell, :inline => "yum-config-manager --disable oscar"
    goldenclient.vm.provision :shell, :inline => "yum -y install mpi4py-mpich mpi4py-openmpi"
    goldenclient.vm.provision :shell, :inline => "si_prepareclient --server server -y"
  end
  config.vm.define "server" do |server|
    server.vm.provision :shell, :inline => "hostname server", run: "always"
    server.vm.provision "shell" do |s|
      s.inline = $etc_hosts
      s.args   = [serverIP, "server"]
    end
    server.vm.provision "shell" do |s|
      s.inline = $etc_hosts
      s.args   = [goldenclientIP, "goldenclient"]
    end
    nodes.collect.each_with_index do |data, index|
      ip = data[1]
      name = data[0]
      server.vm.provision "shell" do |s|
        s.inline = $etc_hosts
        s.args   = [ip, name]
      end
    end
    server.vm.provision :shell, :inline => $linux_disable_ipv6_permanently
    server.vm.provision :shell, :inline => $linux_disable_ipv6,  run: "always"
    server.vm.provision :shell, :inline => $service_iptables_stop, run: "always"
    server.vm.provision :shell, :inline => $setenforce_0, run: "always"
    server.vm.provision "shell" do |s|
      s.inline = $ifcfg_device
      s.args   = [serverIP, "255.255.255.0", "eth1"]
    end
    server.vm.provision :shell, :inline => "ifup eth1", run: "always"
    # restarting network fixes RTNETLINK answers: File exists
    server.vm.provision :shell, :inline => 'service network restart'
    server.vm.provision :file, source: "~/.vagrant.d/insecure_private_key", destination: "~vagrant/.ssh/id_rsa"
    server.vm.provision :shell, :inline => "cp -rp ~vagrant/.ssh ~root/"
    server.vm.provision :shell, :inline => "yum -y install wget"
    server.vm.provision :shell, :inline => "wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O ~root/.ssh/authorized_keys"
    server.vm.provision "shell" do |s|
      s.inline = $chmod_600_dotssh
      s.args   = ["root"]
    end
    server.vm.provision "shell" do |s|
      s.inline = $chown_dotssh
      s.args   = ["root", "root"]
    end
    server.vm.provision :shell, :inline => "yum -y install dhcp"
    server.vm.provision "shell" do |s|
      s.inline = $etc_sysconfig_dhcpd
      s.args   = ["eth1"]
    end
    server.vm.provision :shell, :inline => $epel6
    server.vm.provision "shell" do |s|
      s.inline = $oscar_release_rhel6
      s.args   = [OSCAR_RELEASE]
    end
    server.vm.provision :shell, :inline => "yum -y install systemconfigurator systemimager-server systemimager-x86_64boot-standard systemimager-x86_64initrd_template"
    server.vm.provision :shell, :inline => "yum -y install yum-plugin-versionlock"
    server.vm.provision :shell, :inline => "yum versionlock systemconfigurator systemimager*"
    # configure pxe server
    server.vm.provision :shell, :inline => "yum -y install atftp-server syslinux xinetd"
    server.vm.provision :shell, :inline => "mkdir /var/lib/tftpboot/pxelinux.cfg"
    server.vm.provision :shell, :inline => "cp -p /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot"
    # Attribute disable needs a space before operator [file=/etc/xinetd.d/tftp]
    server.vm.provision :shell, :inline => "sed -i 's/disable.*=.*/disable = no/' /etc/xinetd.d/tftp"
    server.vm.provision :shell, :inline => "service xinetd start"
    server.vm.provision :shell, :inline => "chkconfig xinetd on"
    # configure dhcpd for systemimager pxe tftp
    server.vm.provision "shell" do |s|
      s.inline = $etc_dhcp_dhcpd_conf
      s.args   =  [serverIP, "192.168.10.0", "255.255.255.0", "192.168.10.254", "192.168.10.101", "192.168.10.103"]
    end
    server.vm.provision :shell, :inline => "service dhcpd start"
    server.vm.provision :shell, :inline => "chkconfig dhcpd on"
    if !ENV.fetch('CLIENT_KEXEC', '').empty?
      server.vm.provision :shell, :inline => "echo -e '\nn' | si_getimage --golden-client goldenclient --image node --ip-assignment dhcp --post-install kexec"
    else
      server.vm.provision :shell, :inline => "echo -e '\nn' | si_getimage --golden-client goldenclient --image node --ip-assignment dhcp --post-install reboot"
    end
    server.vm.provision :shell, :inline => "cp -p /var/lib/systemimager/images/node/etc/systemimager/boot/initrd.img /var/lib/tftpboot/initrd.img.node"
    server.vm.provision :shell, :inline => "cp -p /var/lib/systemimager/images/node/etc/systemimager/boot/kernel /var/lib/tftpboot/kernel.node"
    server.vm.provision :shell, :inline => $tftpboot_pxelinux_cfg_default
    server.vm.provision :shell, :inline => "chown -R nobody.nobody /var/lib/tftpboot"
    server.vm.provision :shell, :inline => "service xinetd restart"
    server.vm.provision :shell, :inline => "si_addclients --hosts node101-node103 --domainname localdomain --script node"
    server.vm.provision :shell, :inline => "service systemimager-server-rsyncd start"
    server.vm.provision :shell, :inline => "si_lsimage"
    # the steps below need to be performed every time a new image is fetched with si_getimage
    # make sure vagrant's public key is used
    server.vm.provision :shell, :inline => "su - vagrant -c 'wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O /var/lib/systemimager/images/node/home/vagrant/.ssh/authorized_keys'"
    # remove goldenclient's IP from ifcfg-eth1
    server.vm.provision :shell, :inline => "sed -i '/IPADDR/d' /var/lib/systemimager/images/node/etc/sysconfig/network-scripts/ifcfg-eth1"
    server.vm.provision :shell, :inline => "sed -i '/NETMASK/d' /var/lib/systemimager/images/node/etc/sysconfig/network-scripts/ifcfg-eth1"
    # and enable dhcp on this interface
    server.vm.provision :shell, :inline => "sed -i '/BOOTPROTO/d' /var/lib/systemimager/images/node/etc/sysconfig/network-scripts/ifcfg-eth1"
    server.vm.provision :shell, :inline => "echo BOOTPROTO=dhcp >> /var/lib/systemimager/images/node/etc/sysconfig/network-scripts/ifcfg-eth1"
  end
end

