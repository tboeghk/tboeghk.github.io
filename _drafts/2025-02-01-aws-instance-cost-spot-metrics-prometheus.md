---
layout: instance/blog-post
title: "Collecting AWS instance cost & spot metrics using Prometheus"
tagline: "know your numbers"
featured: true
image: /assets/images/blog/blog-money.jpg
---

Monitoring vital AWS EC2 instance metrics using [Prometheus](https://prometheus.io/) and the [Node Exporter](https://github.com/prometheus/node_exporter) works perfectly. On the instance the Node Exporter collects and exports machine metrics like CPU usage, load average and IOPS. Prometheus can scrape these metrics from each instance using an `ec2_sd_config`.

But what’s missing are instance-specific metadata like **network throughput**, **current costs** and **spot termination notices**. Those are essential information regarding alerting network interface saturation and spot termination of instances. Here’s how to collect this data as well.

<!--more-->

## Collect AWS cost & spot metrics on each instance

I’ll guide you through the steps necessary to collect AWS metadata, cost and spot metrics on each instance running. Ideally, you wrap this into your favorite infrastructure as code tool like Ansible or Cloud-Init.

1. **Prepare Node Exporter to use the [textfile collector](https://github.com/prometheus/node_exporter#textfile-collector)**: Create a directory where the Node Exporter picks up static files containing additional metrics. Also enable the textfile collector pointing to the directory we created using the `--collector.textfile.directory=/etc/node-exporter`switch.

```bash
# Create a directory for static textfile metrics
mkdir -p /etc/node-exporter
chown node-exporter /etc/node-exporter
```

```bash
[Unit]
Description=Node Exporter

[Service]
User=node-exporter
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/node_exporter \
    --collector.textfile.directory=/etc/node-exporter

[Install]
WantedBy=multi-user.target
```

1. **Place a Bash script to scrape instance metadata from [AWS EC2 instance metadata API](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-categories.html)**: The AWS EC2 instance API provides basic metadata information for the current instance. We use the API to scrape the *instance type* and the *instance life cycle* indicating whether it is a spot instance. For spot instances we’ll also scrape the [spot instance action](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-instance-termination-notices.html#instance-action-metadata). In the event of a spot termination of the current instance, we’ll retrieve a `stop` or `terminate` action. All information is written as a `node_aws_info`metric into  `/etc/node-exporter/node_aws.prom` ready to be picked up from the Node exporter.

```bash
#!/usr/bin/env bash
#
# Collect AWS EC2 metadata for this instance
AWS_REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
INSTANCE_LIFE_CYCLE=$(curl -s http://169.254.169.254/latest/meta-data/instance-life-cycle)
INSTANCE_TYPE=$(curl -s http://169.254.169.254/latest/meta-data/instance-type)

# Collect pricing from ec2.shop
INSTANCE_INFO=$(curl -sL "https://ec2.shop?region=${AWS_REGION}&filter=${INSTANCE_TYPE}" -H 'accept: json' | jq '.Prices[]')

instance_storage=$(echo ${INSTANCE_INFO} | jq -r .Storage)
instance_network=$(echo ${INSTANCE_INFO} | jq -r .Network)
instance_cost_hourly=$(echo ${INSTANCE_INFO} | jq -r .Cost)
instance_cost_monthly=$(echo ${INSTANCE_INFO} | jq -r .MonthlyPrice)
instance_spot_price=$(echo ${INSTANCE_INFO} | jq -r .SpotPrice)

if [ "${INSTANCE_LIFE_CYCLE}" = "spot" ] ; then
    instance_cost_hourly_billed=${instance_spot_price}
else
    instance_cost_hourly_billed=${instance_cost_hourly}
fi

# spot information
SPOT_INSTANCE_ACTION=$(curl -sf http://169.254.169.254/latest/meta-data/spot/instance-action | jq -r .action)

cat << EOF > /etc/node-exporter/node_aws.prom
# HELP node_aws_info AWS specific information to this node
node_aws_info{life_cycle="${INSTANCE_LIFE_CYCLE}",storage="${instance_storage}",network="${instance_network}",cost_hourly="${instance_cost_hourly}",cost_monthly="${instance_cost_monthly}",spot_price="${instance_spot_price}",cost_hourly_billed="${instance_cost_hourly_billed}",spot_action="${SPOT_INSTANCE_ACTION}"} 1
node_aws_costs{life_cycle="${INSTANCE_LIFE_CYCLE}"} ${instance_cost_hourly_billed}
EOF
```

1. Place a crontab file in `/etc/cron.d` that triggers the script above

```bash
# writes additional node_aws_info metrics to node exporter
* * * * * node-exporter /usr/local/bin/node-exporter-instance-info.sh
```

Restart your Node Exporter and you’re all set. Verify the presence of the `node_aws_info` metric via curl.

```bash
curl -s "http://localhost:9100/metrics" | grep "node_aws"
```

## Collect AWS & cost metrics in Prometheus

On the Prometheus side, you need to configure a scrape job that scrapes every EC2 instance on the Node Exporter port `9100`. For proper labeling attach the `instance_id`, `architecture`, `availability_zone` and `instance_type` labels via Prometheus relabeling.

```yaml
scrape_configs:
  - job_name: 'nodes'
    metrics_path: /metrics
    ec2_sd_configs:
      - region: eu-west-1
        port: 9100
    relabel_configs:
      - source_labels: [__meta_ec2_instance_state]
        regex: stopped
        action: drop
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id
      - source_labels: [__meta_ec2_architecture]
        target_label: architecture
      - source_labels: [__meta_ec2_availability_zone]
        target_label: availability_zone
      - source_labels: [__meta_ec2_instance_type]
        target_label: instance_type
```

## Craft Prometheus queries & alerts

- Example joins with other node metrics

![alt](/assets/images/blog/aws-metrics.png)

### Spot instance termination alert

```yaml
- alert: AwsSpotInstanceTerminated
  expr: max_over_time(node_aws_info{life_cycle="spot", spot_action!=""}[1h]) > 0
  for: 0m
  labels:
    severity: warning
  annotations:
    summary: The spot instance {{ $labels.instance }} has been terminated.
    description: |
			The spot instance {{ $labels.instance }} reported {{ $labels.spot_action }},
			which means it has been terminated.
```

## Wrap up

The data collected does not represent your whole AWS bill. Other infrastructure costs as traffic, load balancer and non EC2 services are missing. But it gives you good overview of your current EC2 instance costs.
