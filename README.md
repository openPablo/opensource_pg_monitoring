# Open Source Monitoring, full setup

De volgende stack word gebruik:

- Prometheus  om metrics te verzamelen
- Grafana voor het dashboard
- Alertmanager voor alerts te beheren en notificaties te verzenden
- (optioneel) nginx voor reverse proxy
- (optioneel) TimescaleDB + Postgres om historiek bij te houden 

## Prometheus Install

vi /etc/hosts

```
  192.168.56.101 pg12a
  192.168.56.102 pg12b
  192.168.56.103 pg12c
  192.168.56.100 sol
```

#Installing prometheus

```
mkdir /etc/prometheus
mkdir /opt/prometheus
mkdir /data
useradd prometheus
chown -R prometheus:prometheus /etc/prometheus
chown -R prometheus:prometheus /opt/prometheus
chown -R prometheus:prometheus /data
cd /opt/prometheus
```

Copy the correct link for your host type: https://github.com/prometheus/prometheus/releases

```
yum install wget -y
wget ____.tar.gz
tar xzvf _____
mv prometheus-2.13.1.linux-amd64/* .
rm -rf prometheus-2.13.1.linux-amd64
mv /opt/prometheus/prometheus.yml /etc/prometheus/prometheus.yml
```

By writing the targets in a json file instead of writing it directly in prometheus.yml, we can change the targets without restarting prometheus.

vi /etc/prometheus/prometheus.yml

```yaml
cat <<EOT > /etc/prometheus/prometheus.yml
global:           
  scrape_interval:     15s 
  evaluation_interval: 15s                                                                            
alerting:                                             
  alertmanagers:
    - scheme: http                                                 
      path_prefix: /alerts                                                                 
      static_configs:                                                                    
        - targets:                                                                         
          - 127.0.0.1:9093                                                                     
rule_files:                                                                               
  - /etc/prometheus/alert.rules.yml                                                       
scrape_configs:                                                    
  - job_name: 'dummy'  # This is a default value, it is mandatory. 
    file_sd_configs:                                               
      - files:                                                     
        - /etc/prometheus/targets.json
EOT

```

vi /etc/prometheus/targets.json

```json
cat <<EOT > /etc/prometheus/targets.json
[
  {
    "targets": [ "localhost:9100"],
    "labels": {
      "env": "monitoring",
      "job": "node"
    }
  }
]
EOT

```

#moet zien of dit te fixen is zonder als root te runnen
vi /etc/systemd/system/prometheus-server.service

```
[Unit]
Description=Prometheus Server v2.13
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=root
Restart=on-failure
ExecStart=/opt/prometheus/prometheus --config.file=/etc/prometheus/prometheus.yml --web.external-url=https://localhost:/promconsole

[Install]
WantedBy=multi-user.target
Alias=prometheus.service
```

vi /etc/selinux/config

```
SELINUX=permissive
```

```bash
systemctl daemon-reload
systemctl start prometheus-server.service
systemctl enable prometheus-server.service
systemctl status prometheus-server.service
```



## Installing grafana

#https://grafana.com/docs/installation/rpm/

vi /etc/yum.repos.d/grafana.repo

```
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

```bash
yum install grafana -y
firewall-cmd --zone=public --add-port=3000/tcp --permanent
firewall-cmd --reload
grafana-cli plugins install grafana-piechart-panel
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server

```

browse to http://192.168.56.100:3000/login

login,  pw: admin:admin

config: https://prometheus.io/docs/visualization/grafana/

- Node Details: https://grafana.com/grafana/dashboards/10879
- Node Aquarium: https://grafana.com/grafana/dashboards/405
- For postgres: https://grafana.com/grafana/dashboards/6742
- For alertmanager: https://grafana.com/grafana/dashboards/8010

## Installing alertmanager

#Download
https://github.com/prometheus/alertmanager/releases
#scp en untar in /opt/prometheus

```bash
cp alertmanager /usr/sbin/alertmanager
cp amtool /usr/sbin/amtool
mkdir -p /etc/alertmanager
mv alertmanager.yml /etc/alertmanager/altermanager.yml
chown -R prometheus:prometheus /etc/alertmanager/

```

vi /etc/systemd/system/alertmanager.service

```
[Unit]
Description=Alertmanager
Documentation=https://prometheus.io/docs/alerting/configuration/
After=network-online.target

[Service]
User=prometheus
Restart=on-failure
ExecStart=/usr/sbin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml --web.external-url=https://localhost:/alerts   

[Install]
WantedBy=multi-user.target

```

```bash
systemctl daemon-reload
systemctl enable alertmanager
systemctl start alertmanager

#firewall open doen als je wilt testen
#firewall-cmd --zone=public --add-port=9093/tcp
#firewall-cmd --reload

```

Als je de alert.rules wilt valideren:

```
/opt/prometheus/promtool check rules /etc/prometheus/alert.rules.yml
```

vi /etc/alertmanager/alertmanager.yml

```yaml
cat <<EOT > /etc/alertmanager/alertmanager.yml
global:
  resolve_timeout: 1m

route:
  group_by: ['env','node']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 6h
  receiver: standby

receivers:

- name: standby
  email_configs:
  - to: 'pablo.hendrickx@exitas.be'
    send_resolved: false
    from: 'alert@hendrickx.dev'
    smarthost: 'smtp.eu.mailgun.org:587'
    auth_username: 'alert@hendrickx.dev'
    auth_password: '068eec2d4bdbee3a64aea1327b52a72d-09001d55-c3994a38'
    auth_identity: 'alert@hendrickx.dev'

inhibit_rules:

  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
EOT

```

```bash
grafana-cli plugins install camptocamp-prometheus-alertmanager-datasource
service grafana-server restart

```

Install the dasboard grafana.com/dashboards/8010. Create a new datasource, select the prometheus-alertmanager datasource, configure and save.
Add a new dasboard, select import and provide the ID 8010, select the prometheus-alertmanager datasource and save. You should see the following (more or less):

vi /etc/prometheus/alert.rules.yml
Vul in met rules van hier:
https://awesome-prometheus-alerts.grep.to/rules.html
Zie alert.rules.yml in deze dir



## Installing nginx

Optioneel, maar in deze tutorial zijn de settings van prometheus.yml en de systemd service van prometheus en alertmanager aangepast zodat het zou samenwerken met nginx.

#https://devconnected.com/how-to-setup-grafana-and-prometheus-on-linux/

```
sudo yum install epel-release -y
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx

firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=https
firewall-cmd --zone=public --remove-port=3000/tcp --permanent
sudo firewall-cmd --reload

```

vi /etc/nginx/nginx.conf

```
 location /grafana/ {
         proxy_pass http://localhost:3000/;
 }
 location /promconsole {
         auth_basic           "Prometheus Console";
         proxy_pass http://localhost:9090;
 }
 location /alerts {
         auth_basic           "Alertmanager Console";
         proxy_pass http://localhost:9093;
 }

```

systemctl restart nginx
vi /etc/grafana/grafana.ini

```
domain = foo.bar
root_url = %(protocol)s://%(domain)s/grafana/

```

```bash
yum install httpd-tools
htpasswd -c /etc/prometheus/.credentials admin
htpasswd -c /etc/alertmanager/.credentials admin

```

In grafana: data source voor alertmanager aanpassen. url 

van "http://192.168.56.100:9093/#/alerts" naar  "http://192.168.56.100/alerts/#/alerts"

##### Enabling HTTPS on the reverse proxy

Wanneer je een echte DNS gebruikt voor een server, niet voor lokaal:

```
yum install certbot-nginx

certbot –nginx -d linuxtechlab.com -d www.linuxtechlab.com

certbot –nginx -d linuxtechlab.com -d www.linuxtechlab.com

```

#Indien lokaal, en wilt self signen:

#https://www.humankode.com/ssl/create-a-selfsigned-certificate-for-nginx-in-5-minutes

vi /etc/grafana/grafana.ini

```
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        ssl_certificate /etc/ssl/certs/localhost.crt;
        ssl_certificate_key /etc/ssl/certs/localhost.key;
        ssl_protocols TLSv1.2 TLSv1.1 TLSv1;

```

## Timescaledb Install

Optioneel als je een test demo wilt doen, verplicht als je productie omgeving wilt maken.

#### a) TimescaleDB

#https://docs.timescale.com/latest/getting-started/installation/rhel-centos/installation-yum

```bash
yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm -y
yum install postgresql11-server postgresql11 -y postgresql11-contrib
/usr/pgsql-11/bin/postgresql-11-setup initdb
systemctl enable postgresql-11
systemctl start postgresql-11
systemctl status postgresql-11

```



```
sudo tee /etc/yum.repos.d/timescale_timescaledb.repo <<EOL
[timescale_timescaledb]
name=timescale_timescaledb
baseurl=https://packagecloud.io/timescale/timescaledb/el/7/\$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/timescale/timescaledb/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOL

```

```bash
sudo yum update -y
sudo yum install -y timescaledb-postgresql-11

```


vi ~/.bashrc

```bash
PATH=$PATH:/usr/pgsql-11/bin/

```

#reconnect zodat de postgres path is geladen in de root user

```bash
sudo timescaledb-tune
su postgres

```

vi ~/.bashrc

```bash
  PATH=$PATH:/usr/pgsql-11/bin/

```

#### b) Installing pg_prometheus

#https://github.com/timescale/pg_prometheus
#als root

```bash
yum install centos-release-scl-rh -y
yum install -y gcc devtoolset-7 llvm-toolset-7

```



##### Warning: als je een bug krijgt in 'make install' is dit de workaround

#De andere optie is een docker image gebruiken

```bash
mkdir -p /usr/lib64/llvm5.0
mkdir -p /usr/lib64/llvm5.0/bin
cd /usr/lib64/llvm5.0/bin
ln -s /opt/rh/llvm-toolset-7/root/usr/bin/llvm-lto

```

##### einde workaround

```bash
cd /tmp
wget https://github.com/timescale/pg_prometheus/archive/master.zip
unzip master.zip
cd pg_prometheus-master
make
make install

```



#### c) Activating 3 extensions and starting PG

Voeg dit lijntje toe VOOR de andere records in pg_hba. (Pass IP addr aan met IP van prometheus host)

vi /var/lib/pgsql/11/data/pg_hba.conf 

```
host promstore prometheus 127.0.0.1/32 md5
local promstore prometheus  md5

```

vi /var/lib/pgsql/11/data/postgresql.conf

```
  listen_addresses = '*'
  shared_preload_libraries = 'timescaledb, pg_stat_statements, pg_prometheus'

```

```sql
systemctl restart postgresql-11
systemctl status postgresql-11
su postgres
psql
create database promstore;
\c promstore
create extension IF NOT EXISTS  pg_stat_statements;
create extension IF NOT EXISTS  timescaledb;
create extension IF NOT EXISTS  pg_prometheus;
CREATE ROLE prometheus WITH LOGIN PASSWORD 'secret';
GRANT ALL ON SCHEMA prometheus TO prometheus;
\c promstore prometheus
SELECT create_prometheus_table('metrics',use_timescaledb=>true);

```



#### d) Installing prometheus-postgresql-adapter

#https://github.com/timescale/prometheus-postgresql-adapter
cd /opt/prometheus
wget https://github.com/timescale/prometheus-postgresql-adapter/releases/download/v0.6.0/prometheus-postgresql-adapter-0.6.0-linux-amd64.tar.gz
tar xzvf prometheus-postgresql-adapter-0.6.0-linux-amd64.tar.gz
vi /etc/systemd/system/prometheus-adapter.service

```
[Unit]
Description=Prometheus Adapter Service
Documentation=https://github.com/timescale/prometheus-postgresql-adapter
After=network-online.target

[Service]
User=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/prometheus-postgresql-adapter -pg-password=secret -pg-database=promstore -pg-user=prometheus -pg-host=127.0.0.1 -pg-prometheus-log-samples

[Install]
WantedBy=multi-user.target
Alias=prom-adapt.service

```



```
systemctl daemon-reload
systemctl start prometheus-adapter
systemctl enable prometheus-adapter
systemctl status prometheus-adapter

```

vi /etc/prometheus/prometheus.yml

```yaml
remote_write:
  - url: "http://127.0.0.1:9201/write"
remote_read:
  - url: "http://127.0.0.1:9201/read"

```

firewall-cmd --zone=public --add-port=9201/tcp --permanent
firewall-cmd --reload
