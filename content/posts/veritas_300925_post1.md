---
title: "Arctera[Veritas]  Storage Foundation : Identify LUN of DMP ( Dynamic Multipath) device."
series: [veritas]
date: 2025-10-01
draft: false
weight: 10
tags: [Veritas]
categories: [Linux, Veritas]
ShowToc: false
---

This article shows you how to identify LUN ID that maps to a Arctera[Veritas] Storage Foundation  DMP ( Dynamic Multipath).

Steps
-----
###### 1)  List fibre cards and encloser details for future reference. 

	From your Linux shell

	# systool -c fc_host -v |egrep "Class Devic|port_name|port_state|port_id"
	Class Device = "host11"
	Device path = "/sys/devices/pci0000:00/0000:00:04.0/0000:08:00.0/host11"
	port_id = "0x000000"
	port_name = "0x210000e08b8068ae"
	port_state = "LinkDown"									
	Class Device = "host13"
	Device path = "/sys/devices/pci0000:00/0000:00:04.0/0000:08:00.0/host13"
	port_id = "0x000000"
	port_name = "0x210000e08b8068ae"
	port_state = "Online"	
	Class Device = "host14"
	Device path = "/sys/devices/pci0000:00/0000:00:04.0/0000:08:00.0/host10"
	port_id = "0x000000"
	port_name = "0x210000e08b8068ae"
	port_state = "Linkdown"					
	Class Device = "host3"
	Device path = "/sys/devices/pci0000:00/0000:00:04.0/0000:08:00.0/host3"
	port_id = "0x000000"
	port_name = "0x210000e08b8068ae"
	port_state = "Online"

 	#xmdmpadm listenclosure all
	ENCLR_NAME	ENCLR_TYPE	ENCLR_SN	Status 		ARRARY_TYPE	LUN COUNT	FIRMWARE
	emc1		EMC		000297800422	CONNECTED	VMAX-A/A	14		5978
	disk		Disk 		DISKS		CONNECTED	Disk		1		5.26

###### 2) Identify dmpnodename

	# df -h |grep cluster

	# /dev/vx/dsk/RS_cluster_T2/lv_cluster    32GB 1.4G 31 5% /cluster

 	# vxdg list RS_cluster_T2
 	  Group: RS_cluster_T2
 	  dgid : 1716266100.420.hkl2135435
 	  import-id : 1024.41	
 	  version: 290
	  aligment : 8192 (bytes)
	  local_activation: read-write
	  ssb: on
	  autotagging : on
	  copies: nconfig=default nlog=default
	  config: seqno=0.1158 permlen=51360 free=51360 templen=4 loglen=4096
 	  config disk emc1_16f9 copy 1 len=51360 state=clean online
 	  log disk emc1_16f9  copy 1 len=4096

###### 3) Use vxdmpadm getsubpaths to identify the OS devices that represents the paths to your LUN.

	# vxdmpadm getsubpaths dmpnodename=emc1_16f9	#	( dmpnodename DEVICE)
	NAME	STATE[A]	PATH-TYPE[M]	CTL_NAME	ENCLR_TYPE	ENCLR_NAME	ATTRS	Priority
	sdav	ENABLED[A]	-		c3		EMC		emc1		-	-
	sdbj	ENABLED[A]	-		c13		EMC		emc1		-	-
	sdl	ENABLED[A]	-		c13		EMC		emc1		-	-
	sdz	ENABLED[A]	-		c3		EMC		emc1		-	-

###### 3) Either use 

	#ls -ld /sys/block/sd*/device |grep -e sdz -e sdl -e sdbj -e sdav
												 *      ^
	lrwxrwxrwx.	1	root	root	0	Aug 21	14:47	/sys/block/sdav/device	../../../13:0:2:3 
	lrwxrwxrwx.	1	root	root	0	Aug 21  14:47	/sys/block/sdaj/device  ../../../13:0:3:3
	lrwxrwxrwx.	1	root	root	0	Aug 21  14:47	/sys/block/sdl/device	../../../3:0:2:3
	lrwxrwxrwx.	1	root	root	0	Aug 21  14:47	/sys/block/sdz/device	../../../3:0:3:3  **

	Key *  Each host id 13, and 3 ( See systool outout )  Class Device = "host13", and Class Device = "host3"
    	    ^  LUN ID

	#sg_map -x -sd |grep sdz
	/dev/sg25 3 0 3 3 0 /dev/sdz  
	#sg_map -x -sd |grep sdav
	/dev/sg25 13 0 2 3 0 /dev/sdav
	etc for sdl and sdbj

	Notes, on sg_map -x
	after each active sg device name is displayed there are five digits: <host_number> <bus> <scsi_id> <lun> <scsi_type> e.g. 3 0 3 3 0, and 13 0 2 3 0  in example above.

	or 
	#lsblk -S |grep sdz
	sdz	3:0:3:3	disk EMC 	SYMETRIX 5978 fc	
        	*     ^

	* Host from systools
	^ LUN ID

	Then get fdisk /dev/sdz	Get Disk size. This will also help to find the Lun within the Storage Array ! 

        So you LUN ID is 3, on EMC SYMETRIX array with firmware 5978.

