### prometheus grafana监控系统

| 操作系统  | 配置 | IP            | 主机名       |
| --------- | ---- | ------------- | ------------ |
| CentOS7.9 | 2C4G | 192.168.10.21 | prometheus   |
| CentOS7.9 | 2C4G | 192.168.10.22 | grafana      |
| CentOS7.9 | 2C4G | 192.168.10.23 | node         |
| CentOS7.9 | 2C4G | 192.168.10.24 | alertmanager |

```bash
cat >> /etc/hosts << EOF
192.168.10.21 prometheus
192.168.10.22 grafana
192.168.10.23 node
192.168.10.24 alertmanager
EOF
```

```bash
tar -zxvf prometheus-2.26.0.linux-amd64.tar.gz -C /usr/local
mv /usr/local/prometheus-2.26.0.linux-amd64/ /usr/local/prometheus

groupadd prometheus
useradd -g prometheus -s /sbin/nologin prometheus

mkdir -p /var/lib/prometheus
chown -R prometheus /var/lib/prometheus

chown -R prometheus:prometheus /usr/local/prometheus/

vi /usr/local/prometheus/prometheus.yml
```

```bash
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'node'
    scrape_interval: 10s
    static_configs:
    - targets: ['192.168.207.166:9100']
      labels:
        instance: node

```

```bash
/usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml
```

**node**

```bash
tar zxvf node_exporter-1.1.2.linux-amd64.tar.gz
mv node_exporter-1.1.2.linux-amd64 node_exporter

mkdir -p /usr/local/prometheus_exporter
mv node_exporter /usr/local/prometheus_exporter/


/usr/local/prometheus_exporter/node_exporter/node_exporter
```

**grafana**

```bash
tar -zxvf grafana-7.5.6.linux-amd64.tar.gz
mv grafana-7.5.6 /usr/local/grafana
```

```bash
useradd -s /sbin/nologin -M grafana
mkdir -p /data/grafana
chown -R grafana:grafana /usr/local/grafana/ 
chown -R grafana:grafana  /data/grafana/
```

```bash
vi /usr/local/grafana/conf/defaults.ini
```

```bash
#15 data = /data/grafana/data
#21 logs = /data/grafana/log
#24 plugins = /data/grafana/plugins
#27 provisioning = /data/grafana/conf/provisioning
```

```bash
/usr/local/grafana/bin/grafana-server -homepath /usr/local/grafana/
```

**alertmanager**

```bash
tar zxvf alertmanager-0.21.0.linux-amd64.tar.gz
mv alertmanager-0.21.0.linux-amd64 alertmanager
mv alertmanager /usr/local/

```

```
vi /usr/local/alertmanager/alertmanager.yml
```

```bash
global:
  resolve_timeout: 5m
  smtp_from: 'listdarling@163.com'
  smtp_smarthost: 'smtp.163.com:25'
  smtp_auth_username: 'listdarling@163.com'
  smtp_auth_password: 'HKRDNTVAAHTWNCTR'
route:
  group_by: ['alertname']
  group_wait: 5s
  group_interval: 5s
  repeat_interval: 5m
  receiver: 'email'
receivers:
- name: 'email'
  email_configs:
  - to: '209530548@qq.com'
    send_resolved: true
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

```
/usr/local/alertmanager/alertmanager --config.file /usr/local/alertmanager/alertmanager.yml

```

**prometheus**

```
vi /usr/local/prometheus/prometheus.yml
```

```bash
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
    - "/data/prometheus/rules/*.yml"
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'node'
    scrape_interval: 10s
    static_configs:
    - targets: ['192.168.10.23:9100']
      labels:
        instance: node
```

```bash
mkdir -p /data/prometheus/rules/

vi /data/prometheus/rules/disk_rules.yml


# groups：组告警
groups:
# name：组名。报警规则组名称
- name: general.rules
  # rules：定义角色
  rules:
  # alert：告警名称。 任何实例5分钟内无法访问发出告警
  - alert: NodeFilesystemUsage
    # expr：表达式。 获取磁盘使用率 大于百分之80 触发
    expr: 100 - (node_filesystem_free_bytes{mountpoint="/",fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"} * 100) > 1
    # for：持续时间。 表示持续一分钟获取不到信息，则触发报警。0表示不使用持续时间
    for: 1m
    # labels：定义当前告警规则级别
    labels:
      # severity: 指定告警级别。
      severity: warning
    # annotations: 注释 告警通知
    annotations:
      # 调用标签具体指附加通知信息
      summary: "Instance {{ $labels.instance  }} ：{{ $labels.mountpoint }} 分区使用率过高" # 自定义摘要
      description: "{{ $labels.instance  }} ： {{ $labels.job  }} ：{{ $labels.mountpoint  }} 这个分区使用大于百分之80% (当前值：{{ $value }})" # 自定义具体描述

```

```bash
/usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml

dd if=/dev/zero of=test bs=1G count=40

```
