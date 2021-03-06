## Getting prepared

Install prerequisites:

```
apt update
apt install apache2 wrk

```

## Installing tools and getting metrics from HTTP server

Start the HTTP server:
```
sudo systemctl start apache2
```

Download and start apache exporter
```
wget https://github.com/Lusitaniae/apache_exporter/releases/download/v0.5.0/apache_exporter-0.5.0.linux-amd64.tar.gz
tar zxf apache_exporter-0.5.0.linux-amd64.tar.gz
cd apache_exporter-0.5.0.linux-amd64/
./apache_exporter
```

Add new `scrape_config` section into `prometheus.yml`
```
  - job_name: 'apache'
    static_configs:
    - targets: ['localhost:9117']
```

And make your Prometheus to reload a configuration:

```
killall -HUP prometheus
```

## Getting Grafana
Download and install the latest Debian package from https://grafana.org/:

```
wget https://dl.grafana.com/oss/release/grafana_6.0.0_amd64.deb
sudo dpkg -i grafana_6.0.0_amd64.deb
```

Start the `grafana-server`
```
sudo systemctl daemon-reload
sudo systemctl start grafana-server
```
Grafana is now available on [http://localhost:3000/](http://localhost:3000/) with these default credentials:
* Username: `admin`
* Password: `admin`

Dashboards to import:
* [Node dashboard](https://grafana.com/dashboards/405)
* [Apache dashboard](https://grafana.com/dashboards/3894)

## Simulating HTTP usage

Run some load with `wrk`
```
wrk -c 50 -d 120 http://127.0.0.1/
```
