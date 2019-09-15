---
layout: instance/blog-post
title: "Logstash: Geoip resolution using a REST api"
tagline: "Consistent geoip lookups in microservice environments"
image: /assets/images/blog/map.jpg
---

Logstash comes pre-packaged with a geoip filter plugin and a [Maxmind Geolite2 database](https://dev.maxmind.com/geoip/geoip2/geolite2/s). 
But when running Logstash in a container environment, updating the
bundled database can be a hazzle. If you need geoip lookups in other systems,
a rest service is the perfect fit to deliver consistent geoip data. 

<!--more-->

#### Geoip API rest service

During my time at [Shopping24](https://www.s24.com) we developed and open-sourced a 
[rest api for geoip information](https://github.com/observabilitystack/geoip-api) 
that is fed by a Maxmind geoip database. The REST service
served not only [Logstash](https://www.elastic.co/guide/en/logstash/current/index.html) 
but also other system in need of geoip information (e.g. fraud detection).

The Maxmind databases are available for free. A more accurate and detailed
version [can be purchased](https://www.maxmind.com/en/geoip2-city). 
The database files are updated weekly, but updating 
them on a bunch of servers was a hazzle. To ease this, I updated the 
project last week:

1. A Docker container is [available on Docker Hub](https://hub.docker.com/r/observabilitystack/geoip-api)
1. The current Maxmind Geolite2 City database is bundled within the container
1. The container is [built weekly with an updated database](https://hub.docker.com/r/observabilitystack/geoip-api/tags). 
   If you use the `latest` tag in your deployments, you're always up to date.

#### Using the Geoip-API in Logstash

We'll set up Logstash in a Docker-Compose environment. Besides a recent
version of Logstash, we launch the __geoip-api_. The Logstash container
get's his configuration (`default.conf`) mounted from the host's disk.
The Logstash container accepts log file data via the _Beats_ protocol on 
port `5044` from _Filebeat_ instances.

```
# docker-compose.yaml
# -------------------------------------------------
version: "3.7"
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:7.3.1
    ports:
      - 5044:5044
    volumes:
      - /etc/logstash/pipeline/default.conf:/usr/share/logstash/pipeline/default.conf:ro
    depends_on:
      - geoip-api
  geoip-api:
    image: observabilitystack/geoip-api:latest
```
#### Enriching log data with geoip data

We'll assume that our Logstash receives Nginx or Apache access logs via the
_Beats_ protocol. We'll parse the log lines and forward the clientip to the
_geoip-api_ service, which returns geo information as json. We'll merge the 
returned JSON with the log message.

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

    # lookup geoip and anonymize for non rfc1819 ip addresses
    if [clientip] and [clientip] !~ /^10\./  {

        # contact the geoip micro service on port 8080
        # and send the extracted clientip as path parameter
        http {
            url => "http://geoip-api:8080/%{clientip}"

            # place the returned json fields unter the
            # geoip field.
            target_body => "geoip"
            target_headers => "geoip_headers"
        }

        # anonymize your clientip to comply with
        # privacy protection
        fingerprint {
            source => "clientip"
            target => "clientip"
            method => "IPV4_NETWORK"
            key => "24"
        }
    }
}

output {
    # push your metrics to elasticsearch or graylog here
}
```

#### Wrap up

That's all the steps it takes to enrich your log files with geo location
data in an easy to maintain environment. The [weekly builds of the geoip-api service](https://travis-ci.org/observabilitystack/geoip-api)
are a  easy way to stay up to date with Maxmind's Geolite databases.
Tools included are:

* [Maxminds Geolite2 database]()
* [Logstash](https://www.elastic.co/guide/en/logstash/current/index.html)
* [Logstash http filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-http.html)
* [Geoip-API service](https://github.com/observabilitystack/geoip-api)
* [Geoip-API service on Docker Hub](https://hub.docker.com/r/observabilitystack/geoip-api)
