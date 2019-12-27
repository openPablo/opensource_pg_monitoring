# Installing Daemons on monitored host

### Node_exporter (host system info)

Installs the node exporter repo, adds port and starts the service.

```bash
curl -Lo /etc/yum.repos.d/_copr_ibotty-prometheus-exporters.repo https://copr.fedorainfracloud.org/coprs/ibotty/prometheus-exporters/repo/epel-7/ibotty-prometheus-exporters-epel-7.repo
yum install node_exporter -y
firewall-cmd --permanent --zone=public --add-rich-rule='
rule family="ipv4"
source address="192.168.56.100"
port protocol="tcp" port="9100" accept'
firewall-cmd --reload
systemctl start node_exporter && systemctl enable node_exporter
```



### Postgres_exporter

Download the correct release for the host:

https://github.com/prometheus/node_exporter/releases

SCP the binary to all the target hosts to /usr/sbin

```
scp postgres_exporter root@pg12a:/usr/sbin/postgres_exporter
```

Run as root

```bash
su postgres
psql -c "
CREATE OR REPLACE FUNCTION __tmp_create_user() returns void as $$
BEGIN
  IF NOT EXISTS (
          SELECT                       -- SELECT list can stay empty for this
          FROM   pg_catalog.pg_user
          WHERE  usename = 'postgres_exporter') THEN
    CREATE USER postgres_exporter;
  END IF;
END;
$$ language plpgsql;

SELECT __tmp_create_user();
DROP FUNCTION __tmp_create_user();

ALTER USER postgres_exporter WITH PASSWORD 'PASSWORDOFUSERPOSTGRES_EXPORTER';
ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter,pg_catalog;

-- If deploying as non-superuser (for example in AWS RDS), uncomment the GRANT
-- line below and replace <MASTER_USER> with your root user.
-- GRANT postgres_exporter TO <MASTER_USER>;
CREATE SCHEMA IF NOT EXISTS postgres_exporter;
GRANT USAGE ON SCHEMA postgres_exporter TO postgres_exporter;
GRANT CONNECT ON DATABASE postgres TO postgres_exporter;

CREATE OR REPLACE FUNCTION get_pg_stat_activity() RETURNS SETOF pg_stat_activity AS
$$ SELECT * FROM pg_catalog.pg_stat_activity; $$
LANGUAGE sql
VOLATILE
SECURITY DEFINER;

CREATE OR REPLACE VIEW postgres_exporter.pg_stat_activity
AS
  SELECT * from get_pg_stat_activity();

GRANT SELECT ON postgres_exporter.pg_stat_activity TO postgres_exporter;

CREATE OR REPLACE FUNCTION get_pg_stat_replication() RETURNS SETOF pg_stat_replication AS
$$ SELECT * FROM pg_catalog.pg_stat_replication; $$
LANGUAGE sql
VOLATILE
SECURITY DEFINER;

CREATE OR REPLACE VIEW postgres_exporter.pg_stat_replication
AS
  SELECT * FROM get_pg_stat_replication();

GRANT SELECT ON postgres_exporter.pg_stat_replication TO postgres_exporter;"
echo DATA_SOURCE_NAME="postgresql://postgres_exporter:PASSWORDOFUSERPOSTGRES_EXPORTER@localhost:5432/postgres?sslmode=disable" > /var/lib/pgsql/pg.env
exit
cat > /etc/systemd/system/postgres_exporter.service << EOF
[Unit]
Description=Postgres Exporter
Wants=network-online.target
After=postgresql-12.service

[Service]
User=postgres
EnvironmentFile=/var/lib/pgsql/pg.env
ExecStart=/usr/sbin/postgres_exporter

[Install]
WantedBy=default.target
EOF
firewall-cmd --permanent --zone=public --add-rich-rule='
rule family="ipv4"
source address="192.168.56.100"
port protocol="tcp" port="9187" accept'
firewall-cmd --reload
systemctl daemon-reload || systemctl start postgres_exporter || systemctl enable postgres_exporter
```


