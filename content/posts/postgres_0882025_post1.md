---
title: "Pgbackrest utilising a Azure Storage Account - Using a SAS Key - Part 1"
series: [postgres]
date: 2025-08-09
draft: false
weight: 10
tags: [postgres]
categories: [databases, postgres]
ShowToc: false
---

This article shows you how to setup pgbackrest to utilise a azure storage account to store backups 
and archived postgres wal logs.

Steps
-----

###### 1) Create Storage Account

First, setup a Azure Storage Account, either using Terraform, ARM Template Azure Cli, or via Azure Portal.

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

Optional settings for the Azure Storage Account to consider.

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
 
###### 2) Create a Azure Service Princple.

	a) From your cloud shell

		spname - Service Princple name.  
		rgname - Resource Group.  
		storaccname = Your Azure Storage Account.  

		az ad sp create-for-rbac --role "Storage Blob Data Contributor" --name <spname> --scopes /subscriptions/<subID>/resourceGroups/<rgname>/providers/Microsoft.Storage/storageAccounts/<storaccname>  

		e.g.  

		az ad sp create-for-rbac --role "Storage Blob Data Contributor" --name spvmpostgres01 --scopes /subscriptions/xxxxxxxx/resourceGroups/rg-mybackup/providers/Microsoft.Storage/storageAccounts/stpostgres  

	b) In the Azure Console, load your Storage Account , on the left side look for 'Access Control ( IAM)' , then  'Role Assignments'. Look for your service princple, check "Storage Blob Data Contributor" is assigned to your storage account.  

	c) Click on 'Add' in 'Access Control ( IAM)'. Add 'Storage Account Key Operator Service Role' to your service princple.  

	d) Wait for 10mins for Azure to reflect any changes.  


###### 3) Obtain credentials

Once the Storage Account created, obtain a storage details ; account name, container name, endpoint address, and SAS Key.


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
archive-push-queue-max = 2GB 				# Custom to your needs.  
spool-path             = /var/spool/pgbackrest/14  	# Custom to your environment.  
archive-get-queue-max  = 2GB  				# Custom to your needs.  
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

Delete all of your Cluster ( PGDATA, PGLOG , any seperate tablespace locations).     
e.g. 'rm -rf /pg/data'  location of your cluster . 

$ su - postgres  
$ pgbackrest --stanza=sname --type=immediate  --log-level-console=debug restore --link-all

Start Postgres
Check postgres log file for any errors, and 'Execute pg_wal_replay_resume() to promote' If you have
  recommendation to promote issue -  

hostname $ psql
  select pg_wal_replay_resume();


Any Best Practices
---

* Always have a dedicated spool path , and use archive-async so  if there is any delay/error with your Storage account wal files can be temporary archived to the spool directory, and reduce risk of fill up your primary postgres wal directory!


Why Use Azure Storage Accounts
---

*  Azure Storage accounts are ideal for backup data when hosting Postgres in Azure, as backup data is backed up directly to 'different infrastructure', and can be geo-replicated to support out of azure region organisational DR site.
*  Azure Storage accounts only charge you for , Volume of data stored per month + Quantity and types of operations performed, along with any data transfer costs + Data redundancy option selected.  Whereas, using azure managed disk for backups + archives  ( Direct Disk backup ) charge - disk type, backup disk size ( Not volume of backup data), LRS/ZRS,  and typical to meet backup protection requirements would need backup disk snapshoted and backuped up by a azure recovery services vault/Vaeeam. So directly managed disks you effectively pay twice.

Next 
---

[Part 2]({{< ref "posts/postgres_1882025_post1.md" >}}) will look at Azure Storage account SAS Key rotation.
