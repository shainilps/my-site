+++
date = '2026-11-24T00:00:00+05:30'
draft = false
title = 'Postgres: read replica'
tags = ['postgres', 'database']
+++

Just as much as I love OS internals, I also love Databases. Stateful systems such as
these are the reason you dont build distributed system for your backend services.
distributed systems are generally hard, well fair word would be its a tradeoff mostly.
you can't design something perfect for everything. though we are not gonna talk
about distributed systems in this as the title says we are gonna look into postgres
read replica.

I always heard about lot of companies scaling to millions with just a master-slave pattern
with postgres (only effective if you are read heavy system). The concept is very simple
you make mulitple copy of data for two reasons one is as a backup another is reducing the
load on main server by using the replica server as source of information. This is efffective
when you are only reading from replicas instead of writing. when you write to a replica
postgres will reject outright, Lets think that hypothetically it don't then what ? Then what
will happen is pretty simple you get the data consistency problem. you write to two nodes
simultaneously then you have two seperate state conflicting with each other. thats why read
replicas is one effective way to do this. you write to the main server, that sends the data  
to the replicas, replicas just udpates itself and answer the read queries. Though this have a
little tradeoff that is replica may take some time to update its data so read right before that
window will give old data in replica even though the state is updated in main server. that is not
acceptable in most systems. we do have methods to avoid this.

As I said postgres replication is a type of distributed system where there is only single leader
and no built in consensus. What I mean here is suppose your primary server dies then the replicas
will wait forever to recieve something and your databsae will fail for write. where as in consensus
based distributed system will let one of the replicas take over as the leader and will work without any
issue. but again we already have solution to this. people have made systems that will let postgres
work as consensus. one example would be `Patroni`.

Now how does postgres read replica get the data, Postgres usually do operation via Write Ahead Log System.
Well in simple words there are few issues with data systems. suppose you have some transaction that deals
updating a user's name and also his one of his addresses. These both thing can be stored in seperate table
lets say user and address. Now when you do this query the postgres may crash right after updading user
data. If this happens then your data is corrupted. A database system should gurantee this behaviour of
consistency. Inorder to solve this people invented a method called writing operation to a file before doing it.
example instead of doing the operation outright you write to a file as i need to update 01 user's name and his address
afterwards. If the log sucess then that operation is success. you can say that to the client operation is done even
at that point. because postgres will gurantee that the logged operation will happen no matter what. now even if the
postgres crashes it can recover and read the log and write do the operation one more time. basically replaying the log.

This WAL is the one that gets shared across the replicas. You send the WAL to every replicas they replay the log
and find itself as the same postition as the primary. These Log can be two types. One is Logical another is Physical.
You share the log as it is. whatever bytes stored on your log file you just shared it. You dont know what operation that
performs, its upto standby/replica to figure that out. Now logical repliction is another types of replication log.
here the raw operation bytes are not shared as it is. Let say you want to recieve a log of one table's udpates. with
logical replica you can recieive log for that only one file. There is also one more advantage in this that is, suppose
your replica is older version than primary. Now two version can have different encoding scheme so physical replication may
fail here but logical replication wont fail becuase its gives the data as sql operation. Creatively even our app can listen on
this log to do some application level operating when a data gets modified in a certain table instead of polling that table for update
with select query which is slow.

Now let us setup a physical replication with docker compose.

Let us understand some pieces you know inroder to understand the steps:

**LSN**: Log Sequence Number gerenally looks like 16/B374D848. high 32 bits/low 32 bits. first one is the log
file and second one is the offset in that file. each log file is 4 GiB.

**Replication Slot**: there are two types of slot logical and physical. a bookmark on the primary that says
how much farther you have read. it exists because consumer can disconnect and reconnect later resume exaxctly where
it left off.

**Publication**: you don't need publication for physical replication. this exist for logical replication this is
used to control what will you get in logical replication. You can make mutliple type of publication like
publication A of table1, table3 and publication B of table2. This way you seperate out the WAL for the consumer.

Now one more important thing we will learn how do you run a postgres in a replica way ? That was the first question
that came through my mind when I started learning about it. Becuase I wanted to know how does a postgres knows if its
a primary or replica. Well they really made it out of the config, one obivous answer. there is command that let you
copy files and wal from the primary that lets you save the configuration how you connected the primary along with the data
that gets saved. so a config files get written along with data that says what are you and how to connect. the command
is only run once after that you just start the postgres directy as the normal one. The data itself decide that with the config.

A simple physical replication setup of two postgres instance:

`docker-compose.yaml`

```yaml
services: # primary
  postgres:
    image: postgres:17
    container_name: pg-replica-source
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app
    ports:
      - "5433:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-primary.sh:/docker-entrypoint-initdb.d/init-primary.sh:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d app"]
      interval: 2s
      timeout: 3s
      retries: 20

  replica: # standby: read-only copy of the primary
    image: postgres:17
    container_name: pg-replica-standby
    depends_on:
      postgres:
        condition: service_healthy
    user: postgres
    environment:
      PGPASSWORD: repl
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5434:5432"
    # First start: copy the primary, then run as a standby.
    #   -X stream  also stream WAL written during the copy
    #   -R         write standby.signal + primary_conninfo
    #   -C -S      create and use physical slot "replica_slot" on the primary
    entrypoint:
      - bash
      - -c
      - |
        if [ ! -s "$$PGDATA/PG_VERSION" ]; then
          pg_basebackup -h postgres -U repl_user -D "$$PGDATA" -X stream -R -C -S replica_slot
        fi
        exec postgres
    volumes:
      - replica-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d app"]
      interval: 2s
      timeout: 3s
      retries: 20

volumes:
  pgdata:
  replica-data:
```

the `init-primary.sh`

```bash

#!/bin/sh
# Runs once, when the primary's data volume is first created.
set -e

# The user the standby connects as. REPLICATION lets it stream WAL.
psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d "$POSTGRES_DB" \
  -c "CREATE ROLE repl_user WITH LOGIN REPLICATION PASSWORD 'repl';"

# Allow physical replication connections from other containers (the standby).
echo "host replication all all scram-sha-256" >> "$PGDATA/pg_hba.conf"
```

Well one thing I forgot to mention is the `pg_hba.conf`. It's just says how can access the primary that's all.
we are setting it users as all and hosts as all but in production there should be fine grained control
