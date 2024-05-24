Add Prometheus system user and group
```
$ sudo groupadd --system prometheus
$ sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```
Download and install Prometheus MySQL Exporter

```
$ curl -s https://api.github.com/repos/prometheus/mysqld_exporter/releases/latest   | grep browser_download_url   | grep linux-amd64 | cut -d '"' -f 4   | wget -qi -
$ tar xvf mysqld_exporter*.tar.gz
$ sudo mv  mysqld_exporter-*.linux-amd64/mysqld_exporter /usr/local/bin/
$ sudo chmod +x /usr/local/bin/mysqld_exporter
$ mysqld_exporter  --version
```

Create Prometheus exporter database user

```
$ mysql -u root -p

The user should have PROCESS, SELECT, REPLICATION CLIENT grants:

mysql> CREATE USER 'mysqld_exporter'@'localhost' IDENTIFIED BY 'StrongPassword';
mysql> GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqld_exporter'@'localhost';
mysql> FLUSH PRIVILEGES;
mysql> EXIT
```

Configure database credentials

```
# Create database credentials file:
$ sudo vim /etc/.mysqld_exporter.cnf

# Add correct username and password for user create

[client]
user=mysqld_exporter
password=StrongPassword

# Set ownership permissions:
$ sudo chown root:prometheus /etc/.mysqld_exporter.cnf
```

Create systemd unit file ( For Systemd systems )

```
This is for systemd servers, for SysV init system, use Prometheus MySQL exporter init script for SysV init system

Create a new service file:

sudo vi /etc/systemd/system/mysql_exporter.service

Add the following content

[Unit]
Description=Prometheus MySQL Exporter
After=network.target
User=prometheus
Group=prometheus

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/.mysqld_exporter.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=0.0.0.0:9104

[Install]
WantedBy=multi-user.target
If your server has a public and private network, you may need to replace 0.0.0.0:9104 with private IP, e.g. 192.168.4.5:9104
```

When done, reload systemd and start mysql_exporter service

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable mysql_exporter
$ sudo systemctl start mysql_exporter
```

Configure MySQL endpoint to be scraped by Prometheus Server

```
Login to your Prometheus server and Configure endpoint to scrape. Below is an example for two MySQL database servers.

scrape_configs:
  - job_name: server1_db
    static_configs:
      - targets: ['10.10.1.10:9104']
        labels:
          alias: db1

  - job_name: server2_db
    static_configs:
      - targets: ['10.10.1.11:9104']
        labels:
          alias: db2
```

Create / Import Grafana Dashboard for MySQL Prometheus exporter

```
Let’s download MySQL_Overview dashboard which has a good overview of database performance.

$ mkdir ~/grafana-dashboards
$ cd ~/grafana-dashboards
$ wget https://raw.githubusercontent.com/percona/grafana-dashboards/master/dashboards/MySQL_Overview.json

Upload Prometheus MySQL dashboard(s) to grafana
Go to Dashboards > Import > Upload .json file 
Locate the directory with dashboard file and import
Metrics collected should start showing.

You need to restart Grafana server to import these dashboards.

sudo systemctl restart grafana-server
sudo service grafana-server restart
You can then start using the dashboards on Grafana. I’ll do a guide for how to Monitor Linux server with Prometheus, for OS metrics, before then, check similar guides below:

How to monitor Linux systems with Grafana, telegraf, and InfluxDB.
Monitor Linux Server Performance with Prometheus and Grafana in 5 minutes
Monitor Apache Web Server with Prometheus and Grafana in 5 minutes
Your support is our everlasting motiv
```

```
# cat /etc/systemd/system/prometheus.service 

[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
Environment="GOMAXPROCS=1"
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target

# https://computingforgeeks.com/how-to-install-and-configure-prometheus-mysql-exporter-on-ubuntu-18-04-centos-7/
# https://computingforgeeks.com/monitoring-mysql-mariadb-with-prometheus-in-five-minutes/
```
