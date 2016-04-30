-----------
Description
-----------

This is a CentOS systemimager http://systemimager.org/ installation consisting of a server,
golden client, and 3 nodes.
Systemimager has been popular at the beginning of the 21st century for deploying HPC installations,
and used as a base for Open Source Cluster Application Resources (OSCAR) oscar.openclustergroup.org.


----------------------
Configuration overview
----------------------

The golden client is installed first and is used for creating the initial node image.
The golden client image is fetched by the systemimager server (server).
The server is configured to serve dhcp/tftp/pxe to the nodes
on the private 192.168.10.0/24 network (associated with the Vagrant eth1 interfaces).
Note that eth0 is used by Vagrant. Vagrant reserves eth0 and this cannot be currently changed
(see https://github.com/mitchellh/vagrant/issues/2093).
A node (e.g. node101) when booting is installed with the goldenclient image.


                                                   eth0
                  |--------------------------------------------------------------------------
                  |                                                                         |
                  |              192.168.10.5                                               |
                  |          -------------------                 -----------                |
      ----------------  eth0 | server on eth1  |      eth1       | node101 | 192.168.10.101 |
      | Vagrant HOST | ------|                 |-----------------| node102 | 192.168.10.102 |
      ----------------       | dhcp, tftp, pxe | 192.168.10.0/24 | node103 | 192.168.10.103 |
                  |          -------------------                 | ...     | ...            |
                  | eth0     |             |                     |         |----------------|
                  |          | eth0        |                     |         |
             -----------------             | eth1                |         |
          ---| golden client |-----------------------------------|         |
          |  -----------------          192.168.10.0/24          -----------
          |     192.168.10.10                                        |
          |                                                          |
          |-----------------------------------------------------------
                                       eth0


------------
Sample usage
------------

Assuming you have VirtualBox and Vagrant installed
https://www.virtualbox.org/ https://www.vagrantup.com/downloads.html,
disable the default VirtualBox dhcp server (see https://github.com/mitchellh/vagrant/issues/3083):

      $ VBoxManage list dhcpservers
      $ VBoxManage dhcpserver modify --disable --netname HostInterfaceNetworking-vboxnet0
      $ git clone https://github.com/marcindulak/vagrant-systemimager-tutorial-centos6.git
      $ cd vagrant-systemimager-tutorial-centos6
      $ CLIENT_NO_LVM=y CLIENT_KEXEC=y vagrant up goldenclient server

Install the goldenclient image on **node101** and **node102**::

      $ CLIENT_NO_LVM=y vagrant up node101 node102

**Note**: if needed you can monitor the progress of the installation by using the GUI=y
environment variable setting. Be patient, imaging takes a couple of minutes.

Verify the network connectivity on the 192.168.10.0/24 network from one of the nodes::

      $ vagrant ssh node101 -c "sudo su - -c 'ping -I eth1 -c 3 server'"

Verify that mpi4py (an example MPI application), works across the nodes::

      $ vagrant ssh node101 -c "sudo su - vagrant -c 'sshpass -p vagrant ssh -o StrictHostKeyChecking=no node102 hostname'"
      $ vagrant ssh node101 -c '. /etc/profile.d/modules.sh; module load mpich-x86_64; `which mpiexec` -env PYTHONPATH `rpm --eval %python_sitearch`/mpich -iface eth1 --host node101,node102 `rpm --eval %python_sitearch`/mpich/mpi4py/bin/python-mpi -c "from mpi4py import MPI; print MPI.COMM_WORLD.Get_rank()"'

When done, destroy the test machines with::

      $ vagrant destroy -f


------------
Dependencies
------------

1. http://files.brianbirkinbine.com/vagrant-centos-65-x86_64-minimal.box - or any (?)
CentOS 6 box which does not use LVM, as systemimager hangs on installation of images of LVM boxes.


-------
License
-------

BSD 2-clause


--------
Problems
--------

1. installation of an LVM based CentOS 6 box hangs at lvcreate. with:

        lvcreate -L19439616K -n lv_root VolGroup || shellout

   In order to use an LVM goldenclient box do **NOT** set the
   CLIENT_NO_LVM environment variable::

        $ GUI=y CLIENT_KEXEC=y vagrant up goldenclient server node101

2. a (non-LVM) node installed with systemimager hangs with a black VBox screen after reboot.
   Reboot is performed when the CLIENT_KEXEC environment variable is **NOT** set, i.e.
   `--post-install reboot` option to `si_getimage` instead of `--post-install kexec` is used.
   In order to perform this test change Vagrantfile so thr order of boot is first disk, then net
   and enable bioslogodisplaytime. At first boot manually choose LAN pxe boot, the system
   will reboot itself after imaging is finished and hang at boot from disk.

        $ GUI=y CLIENT_NO_LVM=y vagrant up goldenclient server node101

3. updating the goldenclient image (si_prepareclient) and trying to fetch it (si_getimage) fails with:

         rsync: read error: Connection reset by peer (104)
         rsync error: error in rsync protocol data stream (code 12) at io.c(759) [receiver=3.0.6]
         ------------- goldenclient mounted_filesystems RETRIEVAL FINISHED -------------
         Failed to retrieve /etc/systemimager/mounted_filesystems from goldenclient.

   It has been suggested this can be due to rsync requiring reverse DNS lookup
   https://test.sc.fsu.edu/twiki/pub/TechHelp/MorphBank/morphbank-probs.txt
   The suggestion of populating `/etc/hosts` at that page is incorrect.
   First, we need rsync >= 3.1.0 on the goldenclient in order to disable reverse DNS lookups:

         $ vagrant ssh goldenclient -c "sudo su - -c 'yum install -y http://pkgs.repoforge.org/rsync/rsync-3.1.1-1.el6.rfx.x86_64.rpm'"

   Then disable reverse DNS lookups on the goldenclient:

         $ vagrant ssh goldenclient -c "sudo su - -c \"sed -i '/log file/areverse lookup = no' /etc/systemimager/rsyncd.conf\""

   and create the new image with si_prepareclient:

         $ vagrant ssh goldenclient -c "sudo su - -c 'si_prepareclient --server server -y'"

   Same error from si_getimage::

         $ vagrant ssh server -c "sudo su - -c \"echo -e '\n\n\n' | si_getimage --golden-client goldenclient --image node --ip-assignment dhcp --post-install kexec\""


----
Todo
----

1. the configuration of the dhcp server should make use of the list of nodes defined
   on the top of Vagrantfile instead of hard-coding the IPs

2. openmpi exit 1 with no further output

        $ vagrant ssh node101 -c '. /etc/profile.d/modules.sh; module load openmpi-x86_64; `which mpiexec` -x PYTHONPATH=`rpm --eval %python_sitearch`/openmpi --mca plm_rsh_agent "ssh -o StrictHostKeyChecking=no" --mca btl_tcp_if_exclude lo,eth0 --map-by slot --host node101,node102 `rpm --eval %python_sitearch`/openmpi/mpi4py/bin/python-mpi -c "from mpi4py import MPI; print MPI.COMM_WORLD.Get_rank()"'  # exit 1 - no error
