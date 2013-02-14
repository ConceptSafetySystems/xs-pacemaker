xs-pacemaker
============

XenServer plug-ins for Pacemaker

We've written a pair of plug-ins for Pacemaker to help create an automatically redundant active/passive pair of XenServers VMs using a DRBD backend.

This includes two new Pacemaker resource management agents:
XenServerPBD - Monitors and controls plug-in/un-plug of the storage SR/PBD
XenServerVM - Monitors and controls shutdown/start-up of a VM

These plug-ins were derived from the Xen resource agent that comes with Pacemaker.

Full installation on XenServer 6.1
--------------------

NOTE: uname -a on your XenServer should return exactly "2.6.32.43-0.4.1.xs1.6.10.734.170748xen" to use our DRBD RPMs.

We provide a few other RPM versions for older XenServers as well: http://download.locatrix.com/drbd

The below uses "node1.mydomain" and "node2.mydomain" to represent the two nodes.

### Install Pacemaker
`rpm -Uvh http://dl.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm`
`wget -O /etc/yum.repos.d/pacemaker.repo http://clusterlabs.org/rpm/epel-5/clusterlabs.repo`

### Install DRBD Pacemaker plug-ins
`wget http://download.locatrix.com/drbd/xenserver6.1.0/drbd-xen-8.4.3-2.i386.rpm`
`wget http://download.locatrix.com/drbd/xenserver6.1.0/drbd-pacemaker-8.4.3-2.i386.rpm`
`wget http://download.locatrix.com/drbd/xenserver6.1.0/drbd-heartbeat-8.4.3-2.i386.rpm`

`rpm -i drbd-heartbeat-8.4.3-2.i386.rpm`

Installs:
* /etc/ha.d/resource.d/drbddisk
* /etc/ha.d/resource.d/drbdupper
* /usr/share/man/man8/drbddisk.8.gz

`rpm -i drbd-xen-8.4.3-2.i386.rpm`

Installs:
* /etc/xen/scripts/block-drbd

`rpm -i drbd-pacemaker-8.4.3-2.i386.rpm`

Installs:
* /usr/lib/drbd/crm-fence-peer.sh
* /usr/lib/drbd/crm-unfence-peer.sh
* /usr/lib/ocf/resource.d/linbit/drbd

`sed -i -e "s/enabled=0/enabled=1/" /etc/yum.repos.d/CentOS-Base.repo`
`sed -i -e "s/enabled=1/enabled=0/" /etc/yum.repos.d/Citrix.repo`
`yum install -y pacemaker corosync heartbeat`

`chkconfig drbd off`

### Patch XenServer shutdown script
This fixes a race condition I found because of xapi-domains shutdown that calls  /opt/xensource/libexec/shutdown
This was shutting down VMs and then heartbeat was restarting them which caused some confusion during reboot

`wget -O /opt/xensource/libexec/shutdown.patch http://download.locatrix.com/pacemaker/shutdown.patch`
`cd /opt/xensource/libexec`
`cp shutdown shutdown.orig`
`patch -p1 < shutdown.patch`

### VM shutdown configuration
This must be set on every HA VM to ensure that shutdown is handled by pacemaker instead of the system
xe vm-param-set other-config:auto_poweroff=false uuid=$UUID

	corosync-keygen
	chown root:root /etc/corosync/authkey
	chmod 400 /etc/corosync/authkey
	scp /etc/corosync/authkey root@node2.mydomain:/etc/corosync/authkey
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


### DRBD configuration

Setup DRBD on both nodes

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

Copy the DRBD configuration to the 2nd node

	scp /etc/drbd.d/global_common.conf root@node2.mydomain:/etc/drbd.d/global_common.conf
	drbdadm adjust all
	chkconfig --level 35 corosync on
	/etc/init.d/corosync start

### Setup Pacemaker agents

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