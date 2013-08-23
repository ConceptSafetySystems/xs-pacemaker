xs-pacemaker
============

XenServer plug-ins for Pacemaker

We've written a pair of plug-ins for Pacemaker to help create an automatically redundant active/passive pair of XenServers VMs using a DRBD backend.

This includes two new Pacemaker resource management agents:
XenServerPBD - Monitors and controls plug-in/un-plug of the storage SR/PBD
XenServerVM - Monitors and controls shutdown/start-up of a VM

These plug-ins were derived from the Xen resource agent that comes with Pacemaker.

The idea behind these plug-ins is to let you create a Primary/Secondary DRBD back-end shared between 2 servers that is automatically managed by Pacemaker in case of fail-over.

The DRBD back-end is managed by the existing DRBD Pacemaker agent.

The XenServerPBD agent will then setup the SR on the master.

Finally the XenServerVM agent will start/stop the individual VMs themselves.

We provide some DRBD RPMs compiled for various versions of XenServer: http://download.locatrix.com/drbd

Note: The RPM must exactly match your XenServer version - otherwise you can build it yourself with the Citrix XenServer DDK VM.

Full Installation Instructions
--------------------

### DRBD RPMs for XenServer 6.1.0

NOTE: uname -a on your XenServer must return exactly "2.6.32.43-0.4.1.xs1.6.10.734.170748xen" to use these DRBD RPMs.
	wget http://download.locatrix.com/drbd/xenserver6.1.0/drbd-xen-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.1.0/drbd-pacemaker-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.1.0/drbd-heartbeat-8.4.3-2.i386.rpm
	rpm -i drbd-heartbeat-8.4.3-2.i386.rpm
	rpm -i drbd-xen-8.4.3-2.i386.rpm
	rpm -i drbd-pacemaker-8.4.3-2.i386.rpm

#### Patch XenServer 6.1.0 shutdown script
This fixes a race condition I found because of xapi-domains shutdown that calls  /opt/xensource/libexec/shutdown
This was shutting down VMs and then heartbeat was restarting them which caused some confusion during reboot

`wget -O /opt/xensource/libexec/shutdown.patch http://download.locatrix.com/pacemaker/shutdown.patch`
`cd /opt/xensource/libexec`
`cp shutdown shutdown.orig`
`patch -p1 < shutdown.patch`

### DRBD RPMs for XenServer 6.2.0
NOTE: uname -a on your XenServer must return exactly "2.6.32.43-0.4.1.xs1.6.10.734.170748xen" to use our DRBD RPMs.

	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-bash-completion-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-debuginfo-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-heartbeat-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-km-2.6.32.43_0.4.1.xs1.8.0.835.170778xen-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-km-debuginfo-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-pacemaker-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-udev-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-utils-8.4.3-2.i386.rpm
	wget http://download.locatrix.com/drbd/xenserver6.2.0/8.4.3/drbd-xen-8.4.3-2.i386.rpm

	rpm -i drbd-utils-8.4.3-2.i386.rpm
	rpm -i drbd-bash-completion-8.4.3-2.i386.rpm
	rpm -i drbd-udev-8.4.3-2.i386.rpm
	rpm -i drbd-km-2.6.32.43_0.4.1.xs1.8.0.835.170778xen-8.4.3-2.i386.rpm
	rpm -q -a | grep drbd

### DRBD Setup for XenServer 6.1.0 or 6.2.0 (those are the ones I've tested)

I found I had drbd bugs with OVS so I had to do this. `drbdadm primary all` kept hanging for me randomly. This changes OVS to the linux network bridge back-end. Hopefully we won't need this some day
`xe-switch-network-backend bridge`

`reboot`

#### Setup a DRBD cross-over cable
I recommend a direct Gigabit cross-over cable between the 2 servers on a spare NIC. It's not a requirement of course, you just need a connection. Otherwise you can possibly get a split brain situation if the switch power died

Create a network interface for DRBD replication
1. Open up XenCenter
2. Click on the server
3. Select the Networking tab and click the Configure buttom in Management Interfaces
4. Click Add IP Address
`Name: DRBD`
`IP: 10.0.0.3`
`netmask: 255.255.255.0`
`no gateway`
5. OK

Add a hostname and ip address for the x-over network
`vi /etc/hosts`

	10.0.0.3 node1drbd
	10.0.0.4 node2drbd

#### Setup the DRBD volume
Linbit guide section 3.2 explains how to do a setup when you're using a separate hard drive for DRBD. I wanted to use my existing drives only, so this is what I did below.

Also FYI lvm.conf already has a filter for VG_Xen*: `filter = [ "r|/dev/xvd.|", "r|/dev/VG_Xen.*/*|"]`

So that's why I skipped that step from the Linbit guide.

Note PEs free on both servers, pick a common value
`pvdisplay`

Create an identical DRBD volume on both servers for storage. This is an example of what I did
`lvcreate -l 53760 VG_XenStorage-de2c1846-4bf4-83a8-f74e-0bf1d2f10769 -n drdb`
`lvdisplay`

`vi /etc/lvm/lvm.conf`
	write_cache_state = 0
`rm /etc/lvm/cache/.cache`

`cd /etc/init.d`
`wget http://download.locatrix.com/drbd/xenserver6.0.2/lvm`
`chmod 0755 /etc/init.d/lvm`
`chkconfig --add lvm`
`chkconfig lvm on`
`service lvm start`

I recommend trying a reboot now and ensure LVM comes up
`reboot`

Ensure it says available
`lvdisplay`

Ensure the 'on' keywords below must match what "hostname" returns on the servers
`hostname`

#### Setup DRBD configuration
vi /etc/drbd.d/drbd-sr1.res

	resource drbd-sr1 {
		protocol C;

		on node1.mydomain {
			device /dev/drbd1;
			disk /dev/VG_XenStorage-de2c1846-4bf4-83a8-f74e-0bf1d2f10769/drbd;
			address 10.0.0.3:7789;
			meta-disk internal;
		}
		on node2.mydomain {
			device /dev/drbd1;
			disk /dev/VG_XenStorage-30746c9f-2d1f-b6d5-3e3e-1eea2fab2fb1/drdb;
			address 10.0.0.4:7789;
			meta-disk internal;
		}
	}

vi /etc/drbd.d/global_common.conf

	global 
	{ 
		usage-count yes; 
	}
	common 
	{
		protocol C;
		net 
		{
			after-sb-0pri discard-zero-changes;
			after-sb-1pri consensus;
			after-sb-2pri disconnect;
		}
		disk 
		{ 
			fencing resource-only;
		}
		handlers 
		{
			split-brain "/usr/lib/drbd/notify-split-brain.sh"; 
			fence-peer "/usr/lib/drbd/crm-fence-peer.sh";
			after-resync-target "/usr/lib/drbd/crm-unfence-peer.sh";
		}
	}
	
vi /usr/lib/drbd/notify.sh

	Comment out this line at the bottom of the file:
	#echo "$BODY" | mail -s "$SUBJECT" $RECIPIENT
	
	Add these lines to the bottom now:
	HOST_UUID=`xe host-list --minimal`
	xe message-create body="$BODY" host-uuid=$HOST_UUID name=DRBD_ALERT priority=10
	case "$0" in
	*split-brain.sh)
	SUBJECT="DRBD split brain on resource $DRBD_RESOURCE"
	BODY=" Split brain detected, Manual split brain recovery is necessary! "
	# BODY="
	#DRBD has detected split brain on resource $DRBD_RESOURCE
	#between $(hostname) and $DRBD_PEER.
	#Please rectify this immediately.
	#Please see http://www.drbd.org/users-guide/s-resolve-split-brain.html for details on doing so."
	;;

#### Create DRBD resources

On both XenServers execute the following
	drbdadm create-md drbd-sr1
	modprobe drbd
	drbdadm up drbd-sr1

Now ONLY on the XenServer that will be your primary (this will overwrite your secondary server's DRBD data; be careful!)
`drbdadm -- --overwrite-data-of-peer primary drbd-sr1`

Temporarily make the sync speed 1GB
`drbdadm disk-options --resync-rate=1G drbd-sr1`

Wait until full sync finishes
`cat /proc/drbd`

After Full Sync, on Both servers
`chkconfig drbd on`

Run on primary to create the SR
`xe sr-create device-config:device="/dev/drbd1" name-label="DRBD-SR1" type=lvm`

Take note of the SR UUID this outputs

Now move DRBD to the secondary so we can introduce the SR there as well

	PBDUUID=`xe pbd-list device-config:device=/dev/drbd1 params=uuid | awk -F: '{print $2}' | grep -v '^$' |  sed 's/^[ ]//g'`
	echo $PBDUUID
	xe pbd-unplug uuid=$PBDUUID
	drbdadm secondary drbd-sr1
	cat /proc/drbd

On the secondary server

	drbdadm primary drbd-sr1
	cat /proc/drbd
	UUID=<SR UUID FROM ABOVE sr-create command>
	echo $UUID
	vgscan
	HOSTNAME=`hostname`
	echo $HOSTNAME
	HOSTID=`xe host-list params=uuid name-label=$HOSTNAME | awk -F: '{print $2}' | grep -v '^$' |  sed 's/^[ ]//g'`
	echo $HOSTID
	xe sr-introduce uuid=$UUID name-label=DRBD-SR1 type=lvm
	xe pbd-create sr-uuid=$UUID host-uuid=$HOSTID device-config:device=/dev/drbd1

We use Nagios on both servers to monitor DRBD, this is how we setup the NRPE plug-in

	wget --no-check-certificate http://raw.github.com/anchor/nagios-plugin-drbd/master/check_drbd -O /usr/lib/nagios/plugins/check_drbd
	chmod +x /usr/lib/nagios/plugins/check_drbd
	vi /etc/nagios/nrpe.cfg
	#
	command[check_drbd]=/usr/lib/nagios/plugins/check_drbd
	#
	service nrpe restart

NOTE: I wrote a wiki article a while ago explaining how to install NRPE on XCP or XenServer (same thing): http://wiki.xen.org/wiki/NagiosXCP

#### Install Pacemaker on XenServer
`rpm -Uvh http://dl.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm`
`wget -O /etc/yum.repos.d/pacemaker.repo http://clusterlabs.org/rpm/epel-5/clusterlabs.repo`

Installs:
* /usr/lib/drbd/crm-fence-peer.sh
* /usr/lib/drbd/crm-unfence-peer.sh
* /usr/lib/ocf/resource.d/linbit/drbd

`sed -i -e "s/enabled=0/enabled=1/" /etc/yum.repos.d/CentOS-Base.repo`
`sed -i -e "s/enabled=1/enabled=0/" /etc/yum.repos.d/Citrix.repo`
`yum install -y pacemaker corosync heartbeat`

`chkconfig drbd off`

#### Disable VM auto-shutdown
This must be set on every HA VM to ensure that shutdown is handled by Pacemaker instead of the system
`xe vm-param-set other-config:auto_poweroff=false uuid=$UUID`

#### Setup corosync

Initial Corosync setup for both nodes

	corosync-keygen
	chown root:root /etc/corosync/authkey
	chmod 400 /etc/corosync/authkey
	scp /etc/corosync/authkey root@node2.mydomain:/etc/corosync/authkey

Create the Corosync config on both nodes

	vi /etc/corosync/corosync.conf

	totem {
	 version: 2
	 token: 5000
	 token_retransmits_before_loss_const: 20
	 join: 1000
	 consensus: 7500
	 vsftype: none
	 max_messages: 20
	 secauth: off
	 threads: 0
	 clear_node_high_bit: yes

	 interface {
	 ringnumber: 0

	 # changethis!
	 bindnetaddr: 10.0.0.3
	 mcastaddr: 226.94.1.1
	 mcastport: 5405
	 }
	 }

	 logging {
	 fileline: off
	 to_syslog: yes
	 to_stderr: no
	 syslog_facility: daemon
	 debug: on
	 timestamp: on
	 }

	 amf {
	 mode: disabled
	 }

Set corosync to start at boot

	chkconfig --level 35 corosync on
	/etc/init.d/corosync start

#### Setup Pacemaker agents

	crm configure property stonith-enabled="false"
	crm configure property no-quorum-policy=ignore
	crm configure rsc_defaults resource-stickiness=100

	crm
	configure
	primitive drbd0 ocf:linbit:drbd \
	  params drbd_resource=drbd-sr1 \
	  op monitor interval="15" role="Master" \
	  op monitor interval="30" role="Slave" \
	  op start interval="0" timeout="240s" \
	  op stop interval="0" timeout="100s"
	ms ms-drbd0 drbd0 \
	  meta master-max="1" master-node-max="1" clone-max="2" \
	  clone-node-max="1" notify="true"
	commit
	quit


If you want to prefer node 1 when possible add this line:
`crm configure location ms-drbd0-prefer-node1 ms-drbd0 rule 100: node1.mydomain`

Setup the PBD agent

	crm configure
	primitive xs_pbd ocf:locatrix:XenServerPBD params pbd_device="/dev/drbd1" debug="true" \
		op start interval="0s" timeout="60s" \
		op stop interval="0s" timeout="40s" 
	colocation fs_on_drbd inf: xs_pbd ms-drbd0:Master
	order fs_after_drbd inf: ms-drbd0:promote xs_pbd:start
	commit
	bye

Setup the VM agent

	crm configure
	primitive xs_vm ocf:locatrix:XenServerVM params vm_name="dbm" debug="true" \
		op monitor interval="10s" timeout="30s" \
		op start interval="0s" timeout="120s" \
		op stop interval="0s" timeout="240s" 
	colocation xs_vm-with-xs_pbd inf: xs_vm xs_pbd
	order xs_vm-after-xs_pbd inf: xs_pbd:start xs_vm:start
	commit
	bye

Helpful stuff
--------------------

### Temporarily move resource from master
crm resource migrate ms-drbd0 node2.mydomain
crm resource unmigrate ms-drbd0

### Reset everything in Pacemaker

Stop drbd first

	drbdadm disconnect all
	drbdadm down all
	/etc/init.d/drbd stop

Now delete the resources

	crm resource stop ms-drbd0
	crm resource cleanup ms-drbd0
	crm configure delete ms-drbd0

	crm resource stop xs_pbd
	crm resource cleanup xs_pbd
	crm configure delete xs_pbd

	crm resource stop xs_vm
	crm resource cleanup xs_vm
	crm configure delete xs_vm

License
--------------------

Copyright (c) 2012, Locatrix Communications
All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.
Neither the name of Locatrix Communications nor the names of its contributors may be used to endorse or promote products derived from this software without specific prior written permission.
THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.