[![License](https://img.shields.io/badge/license-GPLv3-blue.svg)](https://github.com/Proemion/repmgr-agent/blob/master/LICENSE)

# Mission statement

Postgres clusters managed by [repmgr](http://repmgr.org) use an agent
to ensure application connectivity and proper node fencing.

Since the agent is usually run in a situation where the cluster
is recovering from a node failure, the agent needs to get its
job done quickly and reliably. Our goal is to provide one that
is easy to configure and understand, and defaults to failing
in a safe way when crucial components are not available.

# Key features

* Configuration is stored in the database, so changes are automatically
  replicated across nodes.
* If the database is not available, fails violently to avoid accidentally
  breaking the PGBouncers' configuration fixed by another master node.
* One application server node being down does not prevent the other nodes
  from being updated.
* Small codebase which is easy to understand.

# Installation

Install dependencies:

```
apt-get --no-install-recommends install python-pip python-dev libffi-dev libssl-dev gcc python-psycopg2
pip install spur
```

Create the config table that holds the application servers:

```sql
CREATE TABLE repmgr_<cluster name>.app_servers (
  id SERIAL,
  name     VARCHAR(150),
  ip_addr  VARCHAR(150),
  username VARCHAR(150) DEFAULT 'postgres',
  application_name VARCHAR(150) DEFAULT '',
  enabled  BOOLEAN DEFAULT 't',
  auth_trust BOOLEAN DEFAULT 'f'
);
```

Add application servers to the table:

```sql
INSERT INTO repmgr_<cluster name>.app_servers (name, ip_addr) VALUES
('someserver1', '192.168.123.1'),
('someserver2', '192.168.123.2'),
('someserver3', '192.168.123.3');
```

Allow the postgres user to reload pgbouncer by editing `/etc/sudoers.d/pgbouncer`:

```
postgres ALL = NOPASSWD: /usr/sbin/service pgbouncer reload
```

Configure pgdeploy as repmgr's event_notification_command:

```
event_notification_command = /usr/local/bin/pgdeploy -c <cluster name>
```

Prepare pgbouncer:

```
apt-get install pgbouncer
mkdir /etc/pgbouncer/databases
chown -R postgres. /etc/pgbouncer
```

Configure pgbouncer to include the config file that pgdeploy is going
to create. To do this, edit `/etc/pgbouncer/pgbouncer.ini` like this:

```
[databases]
%include /etc/pgbouncer/databases/<cluster name>.ini
```

Ensure that the `postgres` user on your database servers can SSH to
the `postgres` user on the application servers without being prompted
for a password by exchanging SSH keys.

You can now run `pgdeploy` at any time to update your servers or test
the deployment.

Enjoy!

# Authentication

If you have a lot of database users and you want to avoid listing all
their passwords in the clear in `.pgpass`, you can set the `auth_trust`
flag for specific PGBouncers. Thereby, you delegate authentication
completely to the specified PGBouncer, relying solely on its `userlist.txt`
being correct. For this to work, the PGBouncer instance needs an entry
in `pg_hba.conf` with its authentication method set to `trust`.

# Using separate networks for client access and replication

Normally, pgdeploy uses the conninfo from the `repmgr_<cluster>.repl_nodes`
table to configure the remote pgbouncers. This does not work if you
wish to use a separate backend network for replication, because in
that case, `repmgr_<cluster>.repl_nodes` contains addresses that are
unreachable for your clients. For such a setup, enable the `--public-net`
option and add your masters' public conninfo into a table such as:

```sql
CREATE TABLE repmgr_<cluster>.node_public_info (
  id INTEGER PRIMARY KEY,
  conninfo TEXT
);

INSERT INTO repmgr_<cluster>.node_public_info VALUES
  ( 1, 'host=123.123.123.123 port=5432' );
```

Note that the `id` column cannot be a Foreign Key to the `repl_nodes`
table because repmgr deletes and re-creates records in that table
during `standby register` commands.

To easily view the configuration, you can use this SQL statement:

```sql
SELECT rn.id, rn.name, rn.conninfo, npi.*
FROM repmgr_proemion.repl_nodes rn
INNER JOIN repmgr_proemion.node_public_info npi ON rn.id=npi.id;
```

# Contributing

Discussions, issues and pull requests are always welcome. Please keep
in mind though that we want to keep the codebase as light as possible,
so before sending us a pull request, please get in touch via an issue
first.
