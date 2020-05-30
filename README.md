# PDNS protobuf logger to JSON stream

![](https://github.com/dmachard/pdns_logger/workflows/Publish%20to%20PyPI/badge.svg)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
![PyPI - Python Version](https://img.shields.io/pypi/pyversions/pdns_logger)

| | |
| ------------- | ------------- |
| Author |  Denis Machard <d.machard@gmail.com> |
| License |  MIT | 
| PyPI |  https://pypi.org/project/pdns_logger/ |
| | |

PDNS logger is a daemon in Python 3 that acts a protobuf server for PowerDNS's products.
You can use it to collect DNS queries and responses and to log to syslog or a json remote tcp collector.

## Table of contents
* [Installation](#installation)
* [Execute pdns logger](#execute-pdns-logger)
* [Startup options](#startup-options)
* [Output JSON format](#output-json-format)
* [Systemd service file configuration](#systemd-service-file-configuration)
* [PowerDNS configuration](#powerdns-configuration)
* [Logstash configuration](#logstash-configuration)

## Installation

From pypi, deploy the `pdns logger` with the pip command.
Only Python3 is supported.

```python
pip install pdns_logger
```

After installation, you will have `pdns logger` binary available

## Execute pdns logger 

Two modes exists to execute the `pdns logger`:
 - write output to stdout
 - write output to a remote tcp
 
To run the pdns logger in the first mode, execute the following command without arguments. 
The logger is listening by default on the 0.0.0.0 interface and 50001 tcp port and
DNS queries and responses are also printed directly on stdout in JSON format.

```
# pdns_logger
2020-05-29 18:39:08,579 Start pdns logger...
2020-05-29 18:39:08,580 Using selector: EpollSelector
```

For the second mode, you need to configure the binary with the address of your remote tcp collector.
Start the pdns logger as below for example:

```
# pdns_logger -j 10.0.0.235:6000
2020-05-29 18:39:08,579 Start pdns logger...
2020-05-29 18:39:08,580 Using selector: EpollSelector
2020-05-29 18:39:08,580 Connecting to 10.0.0.235 6000
2020-05-29 18:39:08,585 Connected to 10.0.0.235 6000
```

## Startup options

Command line options are:

```
optional arguments:
  -h, --help  show this help message and exit
  -l L        listen protobuf dns message on tcp/ip address <ip:port>
  -j J        write JSON payload to tcp/ip address <ip:port>
```

## JSON log format

Each events generated by the `pdns-logger` will have the following format:

```json
{
    "dns_message": "AUTH_QUERY",
    "socket_family": "IPv6",
    "socket protocol": "UDP",
    "from_address": "0.0.0.0",
    "to_address": "184.26.161.130",
    "query_time": "2020-05-29 13:46:23.322",
    "response_time": "1970-01-01 01:00:00.000",
    "latency": 0,
    "query_type": "A",
    "query_name": "a13-130.akagtm.org.",
    "return_code": "NOERROR",
    "bytes": 4
}
```

## Systemd service file configuration

System service file for Centos7

```bash
vim /etc/systemd/system/pdns_logger.service

[Unit]
Description=Python protobuf PDNS logger Service
After=network.target

[Service]
ExecStart=/usr/local/bin/pdns_logger -j 10.0.0.2:6000
Restart=on-abort
Type=simple
User=root

[Install]
WantedBy=multi-user.target
```

Activate the service to start on boot

```bash
systemctl daemon-reload
systemctl enable pdns_logger
```

Now start the pdns-logger from systemd

```bash
systemctl start pdns_logger
```

Check the status

```bash
systemctl status pdns_logger
```

## PowerDNS configuration

You need to configure dnsdist or pdns-recursor to active remote logging.
 
### dnsdist

vim /etc/dnsdist/dnsdist.conf

```
rl = newRemoteLogger("10.0.0.97:50001")
addAction(AllRule(),RemoteLogAction(rl))
addResponseAction(AllRule(),RemoteLogResponseAction(rl))
```

Restart the loadbalancer.

### pdns-recursor

vim /etc/pdns-recursor/recursor.conf

```
lua-config-file=/etc/pdns-recursor/recursor.lua
```

vim /etc/pdns-recursor/recursor.lua

```
protobufServer("10.0.0.97:50001", {logQueries=true,
                                   logResponses=true,
                                   exportTypes={'A', 'AAAA',
                                                'CNAME', 'MX', 
                                                'PTR', 'NS',
                                                'SPF', 'SRV',
                                                'TXT'}} )
outgoingProtobufServer("10.0.0.97:50001",  {logQueries=true,
                                            logResponses=true,
                                            exportTypes={'A', 'AAAA',
                                                         'CNAME', 'MX',
                                                         'PTR', 'NS',
                                                         'SPF', 'SRV',
                                                         'TXT'}})
```

Restart the recursor.

## Logstash configuration

With `pdns logger`, you can send DNS logs to a JSON remote collector like logstash.

vim /etc/logstash/conf.d/pdns-logger.conf

```
input {
  tcp {
      port => 6000
      codec => json
  }
}

filter {
  date {
     match => [ "dt_query" , "yyyy-MM-dd HH:mm:ss.SSS" ]
     target => "@timestamp"
  }
}

output {
   elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "pdns-logger"
  }
}
```

Configure your `pdns logger` and restart-it.

Finally, you can have some dashboards on your DNS servers .

![kibana dashboard 1](https://github.com/dmachard/pdns_logger/blob/master/imgs/kibana_dashboard_1.png)

![kibana dashboard 2](https://github.com/dmachard/pdns_logger/blob/master/imgs/kibana_dashboard_2.png)


