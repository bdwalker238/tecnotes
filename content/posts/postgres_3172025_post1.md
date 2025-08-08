---
title: "Postgres pg_service file"
date: 2025-07-31
draft: false
weight: 11
ShowToc: true
---

What is the pg_service file ?
-----------------------------

This article highlights the importance of the Postgres pg_service file (pg_service.conf). The  pg_service file allows libpq connection parameters to be associated with a single service name as part of command argument or when setting PGSERVICE environment variable.  In the Next section , are some examples with s 'service' be used.

Commmand Line examples
----------------------

pg_basebackup --pgdata=$PGDATA --dbname=service=basebackup --verbose --progress --checkpoint=fast --write-recovery-conf --wal-method=stream --waldir=/psql/pglog/13 --create-slot --slot=pgstandby2

$ su - postgres

$ psql service=myservice

postgres@postgres:49010=#

Setting environment variable PGSERVICE
--------------------------------------

pg() { PGSERVICE=$1 psql

}

The advantages of using pg_service.conf are as follows -
---

*  You could have a service name for each database within a Postgres Cluster.  This could be useful for automation.
*  You could create unique service name in conjunction with a unique postgres cluster name as part of an organisation policy. This would allow you to unique identify postgres instances in your organisation/Company for Audit/Configuration Management Purposes.  
*  You only need to update libpq connection parameters in one place.   

How do I set a service entry in pg_service.conf ?   
---

The default **system-wide location of pg_service.conf is identified by 'pg_config --sysconfdir', or environment variable  - PGSERVICEFILE . Once
you have  identified where your  pg_service.conf exists -

$vi  pg_service.conf
{{< line_break >}}
[name]  
host=  
port=  
user=  
passfile=  
{{< line_break >}}

For example,  

[icdlp01]
{{< line_break >}}
host=localhost 
{{< line_break >}}
port=49011 
{{< line_break >}}
user=postgres  
passfile=/psql/home/.pgpass
{{< line_break >}}  
[pgservice]
{{< line_break >}}
host=remotehost  
port=49011  
user=postgres  
passfile=/psql/home/.pgpass
{{< line_break >}}

That's It ..  Any Best Practices  ? 
---

In my experience , I would set one of service name same name as my Postgres cluster name, and for automation have service name enter per database for automation purposes.
