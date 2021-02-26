# Testing PG logical replication

The goal of that PO project is to experiment with the [PostgeSQL Logical Replication](https://www.postgresql.org/docs/12/logical-replication.html) feature in a test environment. Using docker compose, this project deploys 2 DBs, a publisher and a subscriber with some test schema and data.

Logical replication starts by copying a snapshot of the data on the publisher database. Once that is done, changes on the publisher are sent to the subscriber as they occur in real time.

The schema definitions are not replicated, and the published tables must exist on the subscriber. Only regular tables may be the target of replication. For example, you can't replicate to a view.

The tables are matched between the publisher and the subscriber using the fully qualified table name. Replication to differently-named tables on the subscriber is not supported.

Columns of a table are also matched by name. The order of columns in the subscriber table does not need to match that of the publisher. The data types of the columns do not need to match, as long as the text representation of the data can be converted to the target type. For example, you can replicate from a column of type integer to a column of type bigint. The target table can also have additional columns not provided by the published table. Any such columns will be filled with the default value as specified in the definition of the target table.

## Starting the servers

```sh
docker-compose up --build
```

## Accessing the DBs

```sh
# main
psql --user postgres --port 5001 --host localhost netflix
# password: password

# replica
psql --user postgres --port 5002 --host localhost netflix
# password: password
```

## DB Setup

PG requires some specific setup for the main and replica servers. Those options have been set in a same config file for the purpose of that exercise. They should be tweaked according to the documentation and situation.

## Setup publication on main

Here's the specific `postgresql.conf` config:

```conf
wal_level = logical
max_replication_slots = 10 # at least the number of subscriptions expected to connect, plus some reserve for table synchronization
max_wal_senders = 10 # should be set to at least the same as max_replication_slots plus the number of physical replicas that are connected at the same time.
```

Create a publication named `main_publication`:
```SQL
CREATE PUBLICATION main_publication
  FOR ALL TABLES;
```

## Setup subscription on replica

Here's the specific `postgresql.conf` config:

```conf
max_replication_slots = 10 # to at least the number of subscriptions that will be added to the subscriber
max_logical_replication_workers = 4 # at least the number of subscriptions, again plus some reserve for the table synchronization. max_worker_processes may need to be adjusted to accommodate for replication workers, at least (max_logical_replication_workers + 1)
max_worker_processes = 8
max_sync_workers_per_subscription = 2
```

Setup a subscription to the main DB:
```SQL
CREATE SUBSCRIPTION replica_subscription
  CONNECTION 'host=main-db user=postgres password=password dbname=netflix'
  PUBLICATION main_publication;
```

## Monitoring replication

```SQL
SELECT * FROM pg_stat_subscription;
```

## Testing some behaviors

```SQL
INSERT INTO movies (id, title) VALUES (4, 'The Godfather');
INSERT INTO movies (id, title) VALUES (5, 'The Wizard of Oz');
INSERT INTO movies (id, title) VALUES (6, 'Forrest Gump');
INSERT INTO movies (id, title) VALUES (7, '2001: A Space Odyssey');
INSERT INTO movies (id, title) VALUES (8, 'Pulp Fiction');
```
