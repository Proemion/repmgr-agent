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
apt-get install python-pip python-dev libffi-dev libssl-dev
pip install spur cryptography
```

Create the config table that holds the application servers:

```sql
CREATE TABLE repmgr_<cluster name>.app_servers (
  id SERIAL,
  name     VARCHAR(150),
  ip_addr  VARCHAR(150),
  username VARCHAR(150) DEFAULT 'postgres',
  application_name VARCHAR(150) DEFAULT '',
  enabled  BOOLEAN DEFAULT 't'
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

Ensure that the `postgres` user on your database servers can SSH to
the `postgres` user on the application servers without being prompted
for a password by exchanging SSH keys.

You can now run `pgdeploy` at any time to update your servers or test
the deployment.

Enjoy!

# Contributing

Discussions, issues and pull requests are always welcome. Please keep
in mind though that we want to keep the codebase as light as possible,
so before sending us a pull request, please get in touch via an issue
first.
