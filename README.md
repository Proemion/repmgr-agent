# repmgr-agent

repmgr clusters need an agent to update the PGBouncer configuration on the
application servers.

This one stores all its configuration in the Postgres database so that
the configuration is automatically replicated across the cluster.

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
  name    VARCHAR(150),
  ip_addr VARCHAR(150),
  enabled BOOLEAN DEFAULT 't'
);
```

Add application servers to the table:

```sql
INSERT INTO repmgr_<cluster name>.app_servers (name, ip_addr) VALUES
('someserver1', '192.168.123.1'),
('someserver2', '192.168.123.2'),
('someserver3', '192.168.123.3');
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
