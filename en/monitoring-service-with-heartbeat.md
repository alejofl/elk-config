# Service Monitoring with Heartbeat

## Introduction

Heartbeat is an infrastructure and service monitoring tool that enables operations teams to monitor the availability and performance of their services. Heartbeat is part of the Elastic stack.

The goal is to monitor any service or server that can receive a TCP or ICMP request. Heartbeat can be configured to monitor HTTP, HTTPS, TCP, ICMP, etc.

Typically, Heartbeat is installed as part of a monitoring service running on a separate host, possibly even outside the network where the services to be monitored are running.

## Installation on Linux

To install Heartbeat on Linux, we first need to download the installation package. For this, we can go to the Elastic downloads page and copy the download link for the Heartbeat package.

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-8.13.4-amd64.deb
```

Once the package is downloaded, we install it with the following command:

```bash
sudo dpkg -i heartbeat-8.13.4-amd64.deb
```

## Configuration

Heartbeat is configured through a configuration file called `heartbeat.yml`. This file is located in the `/etc/heartbeat` folder.

The important part is to define the output of Heartbeat. Initially, we want the information to be sent directly to Elasticsearch. To do this, we need to configure the Heartbeat output as follows:

```yaml
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["10.1.102.178:9200", "10.1.103.227:9200"]
  # Performance preset - one of "balanced", "throughput", "scale", "latency", or "custom".
  preset: balanced
```

We specify the Elasticsearch hosts to which monitoring data will be sent. In this case, the data is being sent to two Elasticsearch nodes, so if one of them fails, the other can continue to receive data.

## Monitors

There are two ways to define a monitor:

- By adding a `.yml` file to the `/etc/heartbeat/monitors.d` folder.
- By adding a section in the Heartbeat configuration file. The section to add is `heartbeat.monitors`. Then, a list of all monitors is written.

In both cases, the content is the same. Below is an example of an HTTP monitor:

```yaml
type: http
enabled: true
# ID used to uniquely identify this monitor in Elasticsearch even if the config changes
id: webserver-monitor
# Human readable display name for this service in Uptime UI and elsewhere
name: Webserver Monitor
# List of URLs to query
urls: ["http://10.1.1.254"]
# Configure task schedule
schedule: '@every 10s'
# Total test connection and data exchange timeout
timeout: 25s
```

In this case, a web server running at `http://10.1.1.254` is being monitored. The monitor runs every 10 seconds, and the timeout is 25 seconds. If the webserver does not respond within 25 seconds, the monitor will consider it down.

## Starting the Service

To start the Heartbeat service, use the following command:

```bash
sudo systemctl start heartbeat
```

To have the service start automatically when the machine boots up, use the following command:

```bash
sudo systemctl enable heartbeat
```

## Elasticsearch Configuration

For Heartbeat to send data to Elasticsearch, it is necessary to configure the indices to be used. For this, you can use the following command provided by Heartbeat:

```bash
heartbeat setup -e
```

## Visualizing Data in Kibana

Heartbeat comes with pre-built dashboards and UIs to visualize the status of services. The dashboards are available in the [uptime-contrib GitHub repository](https://github.com/elastic/uptime-contrib/blob/master/dashboards/7.x/http_dashboard.ndjson).

To import the dashboards into Kibana, go to the Management section and then to the Saved Objects section.

> [!TIP]
> The Heartbeat configuration file used in the demonstration is available in the repository at `/files/linux/heartbeat.yml`.
