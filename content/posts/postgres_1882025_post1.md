---
title: "Pgbackrest utilising a Azure Storage Account - Using a SAS Key - Part 2"
series: [postgres]
date: 2025-08-18
draft: false
weight: 10
tags: [postgres]
categories: [databases, postgres]
ShowToc: false
---

This article shows you how to Generate the Storage SAS token, and update the PgBackrest Config file.  It
also describes items to consider if you want to use Azure Storage Account using SAS keys
for authentication & Pgbackrest. 

Steps
-----

###### 1) Retrieve the Storage Account key.  

	account_key=$(az storage account keys list \
    	--resource-group "azureresourcegroup" \
    	--account-name "storage_account_name" \
    	--query "[0].value" \
    	--output tsv)
 
###### 2) Generate the Storage SAS token.  

	sas_token=$(az storage container generate-sas \
    	--account-name "storage_account_name" \
    	--name "container_name" \
    	--permissions acdlrw \
    	--expiry "expirydate" \
    	--https-only \
    	--account-key "account_key" \
    	--output tsv)

###### 3) Replace the old token with the new token


	awk -v sas_token="$sas_token" '/^repo1-azure-key=/ {$0="repo1-azure-key=" sas_token} {print}' "$pg_backrest_conf" > /tmp/pgbackrest.conf.tmp && mv /tmp/pgbackrest.conf.tmp "$pg_backrest_conf"



###### 4) Run pgbackrest backup.


su - postgres  
select pg_switch_wal(); select pg_switch_wal(); exit;  
$ pgbackrest --stanza=sname --type=full backup

Check your Postgres log file for errors.


Items to consider
---


* For Azure SAS tokens to be useful with pgbackrest you need to ensure '--expiry "expirydate"' expiry's after
  next Azure Storage Account SAS Token generatation, and ensure you automate rotating the SAS tokens on regular bases. Warning - If you do not do this your backups and/or archiving will Fail. 

* Using Azure Storage Account SAS Tokens can be useful, as ensures the repoX-azure-key in pgbackrest regular changes.

Final thoughts
---

I hope you find  both [Part 1]({{< ref "posts/postgres_0882025_post1.md" >}}) and Part 2 useful. 


