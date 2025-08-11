---
title: "Pgbackrest utilising a Azure Storage Account"
series: [postgres]
date: 2025-08-09
draft: false
weight: 10
tags: [postgres]
categories: [databases, postgres]
ShowToc: false
---

This article shows you how to setup pgbackrest to use a azure storage account to store backups 
and archived postgres wal logs.

Steps
-----

###### 1) Create Storage Account

First you need to setup a Azure Storage Account, either using Terraform, ARM Template Azure Cli, or via Azure Portal.

Mandatory settings for the Azure Storage Account.

Resouce Group - Name of your resource group.
{{< line_break >}}
Storage Account Name - Name of storage account. 
{{< line_break >}}
Region -  Your deployment Region. 
{{< line_break >}}
Primary Service -  Azure Blob Storage or Azure Data Lake Storage Gen 2.
{{< line_break >}}
Enable Storage account key access.

Optional settings for the Azure Storage Account.

Performance - Standard or Premium. 
{{< line_break >}}
Redundancy - Align your organisation RPO and RTO requiements.
{{< line_break >}}
Networking - Align to your organisation network policy , probably a private end point.
{{< line_break >}}
Access tier -  Hot/Cool/Cold.  Hot is a practical choice if you conistently restoring a production backup to uat once a week!

Tip -  I would keep your storage account in a different azure resource group to your postgres VM's,
so if you accidently delete the Postgres resource group + VMS your backup data is seperated. Plus if you kept
your Storage Account in same Azure Resource group that manages your Recovery Services vault(s),  your backup data is in one place.
 
###### 2) Setup VM Managed Identity 

	a) Double-Click on your new Storage Account.
        b) Access Control ( IAM ).
	c) Click 'Add'. 
        d) Click 'Role', Search for 'Storage Blob Data Contributor' in search box, and select it.
	e) Click Next.
	f) Select Managed identity , Click 'Select Members'.
	g) Select Virtual machine for Managed Identity, and hostname(s) of you postgres cluster. **
        h) Review and Assign - Twice. 


** This assumes you had setup a system managed identify for your Postgres VM. If not you need to enable this on your
VM.


###### 3) Obtain credentials

Once the Storage Account created, obtain a storage details ; account name, container name, endpoint address.

###### 3) Populate pgbackrest.conf

In your favourite editor - 

$ su - postgres  
edit your pgbackrest.conf.


[global]  
repo1-type=azure  
repo1-azure-account=storage account name  
repo1-azure-container=storage account container name  
repo1-azure-endpoint=Storage account endpoint address  
repo1-azure-key= Storage account key  
repo1-azure-key-type=sas  
repo1-path=/storage account container name  
start-fast=y  
repo1-retention-full=retention value  
log-level-console=info  
log-level-file=detail  
link-all=y
start-fast=y
archive-async=y  
archive-copy=y  
archive-check=y  
process-max=4  
archive-push-queue-max = 2GB  
spool-path             = /var/spool/pgbackrest/14  
archive-get-queue-max  = 2GB  
[sname]  
pg1-path=PGDATA Dir  
pg1-port=PG PORT  



###### 4) pgbackrest stanza-create

$ su - postgres  
$pgbackrest stanza-create --config pgbackrest.conf --stanza=sname –log-level-console=info  
$pgbackrest --stanza=sname --log-level-console=debug check  
$pgbackrest --stanza=sname --type=full backup  

Hint -- sname = Stanza name in pgbackrest.conf.

###### 5) Backup and Archive 

su - postgres  
$ psql 	# To test archive  
select pg_switch_wal(); select pg_switch_wal(); exit;  
$ pgbackrest --stanza=sname --type=full backup

Check your Postgres log file for errors.

###### 6) Test Backup
Stop Postgres to test a restore 

Delete your Cluster  rm -rf  location of cluster 

$ su - postgres  
$ pgbackrest --stanza=sname --type=immediate  --log-level-console=debug restore --link-all

Start Postgres
Check postgres log file for any errors, and 'Execute pg_wal_replay_resume() to promote' If you have
  recommendation to promote issue -  

hostname $ psql
  select pg_wal_replay_resume();


Any Best Pratices
---

* Always have a dedicated spool path , and use archive-async so  if there is any delay/error with your Storage account wal files can be temporary archive to the spool directory, and reduce risk of fill up your primary postgres wal directory!

Why Use Azure Storage Accounts
---

*  Azure Storage accounts are ideal for backup data when hosting Postgres in Azure, as backup data is backed up directly to 'different infrastructure', and can be geo-replicated to support out of azure region organisational DR site.
*  Azure Storage accounts only charge you for , Volume of data stored per month + Quantity and types of operations performed, along with any data transfer costs + Data redundancy option selected.  Whereas, using azure managed disk for backups + archives  ( Direct Disk backup ) charge - disk type, backup disk size ( Not volume of backup data), LRS/ZRS,  and typical to meet backup protection requirements would need backup disk snapshoted and backuped up by a azure recovery services vault/Vaeeam. So directly managed disks you effectively pay twice.
