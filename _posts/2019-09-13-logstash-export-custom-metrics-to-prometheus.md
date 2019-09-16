---
layout: instance/blog-post
title: "Logstash: exporting custom metrics to Prometheus"
tagline: "mine the gold in your logfiles"
image: /assets/images/blog/logs-winter.jpg
featured: true
---

Logstashs regular expression engine [_Grok_](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html) is the swiss army knife when
it comes to mine gold in your logfiles. Besides pure log enrichment,
metric gathering of log data is a common requirement. Unfortunately Logstash
is lacking a native solution to export custom metrics to [Prometheus](https://prometheus.io/). In 
this post, I'll walk you through a solution utilizing the [_graphite-exporter_](https://github.com/prometheus/graphite_exporter)
as an adapter.

<!--more-->

#### Setup

We'll set up Logstash in a Docker-Compose environment. Besides a recent
version of Logstash, we launch a [_graphite-exporter_](https://github.com/prometheus/graphite_exporter). The _graphite-exporter_
is knifty tool, that accepts Graphite metrics and exports them as Prometheus
metrics on a http port. In Logstash we are going to use the _Graphite_ output 
plugin to send metrics to the _graphite-exporter_ and here's how it's done:

Both containers get their configuration mounted from the host's disk 
(`default.conf` and `graphite-mapping.yaml`). The Logstash container
accepts log file data via the _Beats_ protocol on port `5044` from _Filebeat_
instances. Later we can configure our Prometheus server to harvest metrics
on port `9108`.

```
# docker-compose.yaml
# -------------------------------------------------
version: "3.7"
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:7.3.1
    ports:
      # Filebeat sends here
      - 5044:5044
    volumes:
      - /etc/logstash/pipeline/default.conf:/usr/share/logstash/pipeline/default.conf:ro
    depends_on:
      - graphite-exporter
  graphite-exporter:
    image: prom/graphite-exporter
    command: --graphite.mapping-config=/etc/logstash/graphite-mapping.yaml --graphite.mapping-strict-match
    volumes:
      - /etc/logstash/graphite-mapping.yaml:/etc/logstash/graphite-mapping.yaml:ro
    ports:
      # Prometheus harvests here
      - 9108:9108
```
#### Mining metrics in the Logstash pipeline

We'll assume that our Logstash receives Nginx or Apache access logs via the
_Beats_ protocol. Let's assume we want to expose a Prometheus metric counting 
the http response codes. Below you'll find the Logstash pipeline configured.

```
# default.conf
# -------------------------------------------------
# Accept http access log lines via Beats protocol.
input {
    beats {
        port => 5044
    }
}

filter {

    # Parse the http access log using a predifined
    # Grok pattern.
    grok {
        match => { "message" => "%{COMMONAPACHELOG}" }
    }

    # The metrics filter plugin emits synthetic messages
    # into the Logstash pipeline every 30 seconds.
    metrics {

        # The message will contain these fields x http response
        # code. Fields will e.g. be:
        #  - http_requests_total.200
        #  - http_requests_total.404
        #  - http_requests_total.503
        meter => [
            'http_requests_total.%{response}' 
        ]

        # We're not compute any meter rates as we're only exporting
        # the total count towards Prometheus later.
        rates => []

        # We tag the synthetic message to pick it up in the
        # output section below.
        add_tag => "metric"
        flush_interval => 30
        ignore_older_than => 120
    }
}

output {

    # We send all synthetic metric messages to the graphite exporter
    # we started along Logstash. We'll send all fields of the message
    # as metricss (http_requests_total.200, http_requests_total.404, ...)
    if "metric" in [tags] {
        graphite {
            host => "graphite-exporter"
            port => 9109
            fields_are_metrics => true
        }
    } else {
        # push your metrics to elasticsearch or graylog heres
    }
}
```
#### Configuring Graphite to Prometheus metric mapping

As soon as we start to send http access logs to Logstash, it will
emit Graphite metrics to the _graphite-exporter_ every 30 seconds.
Translating incoming Graphite metrics like `http_requests_total.200.count`
into Prometheus metrics `http_requests_totals{code="200", handler="nginx"}`
can be configured in the configuration file
of the _graphite-exporter_:

```
# graphite-mapping.yaml
# -------------------------------------------------
mappings:
- match: http_requests_total.*.count
  name: http_requests_totals
  labels:
    code: $1
    handler: nginxs
```

#### Wrap up and outlook

Prometheus has a multi dimensional data model. To utilize it, you could add more 
fields to the metrics exported to the _graphite-exporter_ (e.g. `http_requests_total.%{response}.%{verb}.%{hostname}`).
Then, http response codes can be split by incoming hostname or http request verb.

In this small example we managed to gather metrics from http access logs and expose them to 
Prometheus. Tools included are:

* [Logstash](https://www.elastic.co/de/products/logstash)
* [Logstash Grok filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)
* [Logstash metrics filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-metrics.html)
* [Logstash Graphite output plugin](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-graphite.html)
* [_graphite-exporter_](https://github.com/prometheus/graphite_exporter)
