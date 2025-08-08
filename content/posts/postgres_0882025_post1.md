---
title: "Pgbackrest with Azure Storage Account"
date: 2025-08-09
draft: false
weight: 10
ShowToc: true
---

Using pgbackrest with Azure Storage Account 
-------------------------------------------

This article shows you how to setup pgbackrest to use azure storage account. 

Steps
-----

1)

First you need to setup a Azure Storage Account, either using Terraform, ARM Template Azure Cli, or via Azure 

Mandatory Values

Resouce Group - Name of your resource group.
{{< line_break >}}
Storage Account Name - Name of storage account. 
{{< line_break >}}
Region -  Your deployment Region. 
{{< line_break >}}
Primary Service -  Azure Blob Storage or Azure Data Lake Storage Gen 2.
{{< line_break >}}
Enable Storage account key access.

Optional

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
 

2) Once the Storage Account created, obtain a storage account key.

3) In your pgbackrest.conf file , add  

[global]
repo1-type=azure
repo1-azure-account=<Azure Storage Account Name>
repo1-azure-container=<Azure Container Name>
repo1-azure-endpoint=<End Point>
repo1-azure-key=<Key>
repo1-azure-key-type=shared
repo1-path=<Repo Path with Meta Data>
start-fast=y
repo1-retention-full=Number of Full Backups
log-level-console=info
log-level-file=detail
link-all=y
archive-async=y
archive-copy=y
archive-check=y
process-max=4
archive-push-queue-max = 2GB
spool-path             = <Spool Path>
archive-get-queue-max  = 2GB
[stanza name]
pg1-path=PGDATA
pg1-port=PGPORT

4) Then 

pgbackrest stanza-create --config pgbackrest.conf --stanza=<name> –log-level-console=info
pgbackrest --stanza=<name> --log-level-console=debug check
pgbackrest --stanza=<name> --type=full backup

psql - To test archive

select pg_switch_wal()

Check your Postgres log file for errors.

Stop Postgres to test a restore 

Delete your Cluster
pgbackrest --stanza=name --type=immediate  --log-level-console=debug restore --link-all
Start Postgres.

Any Best Pratices
---

* Always have a dedicated spool path , and use archive-async so  if there is any delay /error with your Storage account
  files can be temporary archive to the spool directory, and reduce risk of fill up your primary wal directory!

Why Use Azure Storage Accounts
---

*  Azure Storage accounts are ideal for backup data when hosting Postgres in Azure, as backup data is backed up directly
   to different infrastructure, and can be geo-replicated.
*  Azure Storage accounts only charge you for , Volume of data stored per month + Quantity and types of operations performed, along with any data transfer costs + Data redundancy option selected.  Whereas, using azure managed disk for backups + archives charge - disk type, disk size ( Not volume of backup data), LRS/ZRS,  and typical to meet backup protection requirements would need to be disk snapshoted or backuped up by  Recovery Services vault. So directly managed disks you pay twice.
