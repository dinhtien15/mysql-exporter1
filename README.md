### mysql-exporter1

Add Prometheus system user and group
```
$ sudo groupadd --system prometheus
$ sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```

