---
title: Monitor CockroachDB with Prometheus
summary: How to pull CockroachDB's time series metrics into Prometheus.
toc: true
docs_area: manage
---

CockroachDB generates detailed time series metrics for each node in a cluster. This page shows you how to pull these metrics into [Prometheus](https://prometheus.io/), an open source tool for storing, aggregating, and querying time series data. It also shows you how to connect [Grafana](https://grafana.com/) and [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) to Prometheus for flexible data visualizations and notifications.

## Before you begin

- Make sure you have already started a CockroachDB cluster, either [locally](start-a-local-cluster.html) or in a [production environment](manual-deployment.html).

- Note that all files used in this tutorial can be found in the [`monitoring`](https://github.com/cockroachdb/cockroach/tree/master/monitoring) directory of the CockroachDB repository.

## Step 1. Install Prometheus

1. Download the [2.x Prometheus tarball](https://prometheus.io/download/) for your OS.

2. Extract the binary and add it to your system `PATH`. This makes it easy to start Prometheus from any shell.

3. Make sure Prometheus installed successfully:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ prometheus --version
    ~~~

    ~~~
    prometheus, version 2.34.0 (branch: HEAD, revision: 881111fec4332c33094a6fb2680c71fffc427275)
      build user:       root@d80b449ae319
      build date:       20220315-15:04:36
      go version:       go1.17.8
    ~~~

## Step 2. Configure Prometheus

1. Download the starter [Prometheus configuration file](https://github.com/cockroachdb/cockroach/blob/master/monitoring/prometheus.yml) for CockroachDB:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/prometheus.yml \
    -O prometheus.yml
    ~~~

    When you examine the configuration file, you'll see that it is set up to scrape the time series metrics of a single, insecure local node every 10 seconds:
    - `scrape_interval: 10s` defines the scrape interval.
    - `metrics_path: '/_status/vars'` defines the Prometheus-specific CockroachDB endpoint for scraping time series metrics.
    - `scheme: 'http'` specifies that the cluster being scraped is insecure.
    - `targets: ['localhost:8080']` specifies the hostname and `http-port` of the Cockroach node to collect time series metrics on.

2. Edit the configuration file to match your deployment scenario:

    Scenario | Config Change
    ---------|--------------
    Multi-node local cluster | Expand the `targets` field to include `'localhost:<http-port>'` for each additional node.
    Production cluster | Change the `targets` field to include `'<hostname>:<http-port>'` for each node in the cluster. Also, be sure your network configuration allows TCP communication on the specified ports.
    Secure cluster | Uncomment `scheme: 'https'` and comment out `scheme: 'http'`.

4. Create a `rules` directory and download the [aggregation rules](https://github.com/cockroachdb/cockroach/blob/master/monitoring/rules/aggregation.rules.yml) and [alerting rules](https://github.com/cockroachdb/cockroach/blob/master/monitoring/rules/alerts.rules.yml) for CockroachDB into it:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ mkdir rules
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cd rules
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ wget -P rules https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/rules/aggregation.rules.yml
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ wget -P rules https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/rules/alerts.rules.yml
    ~~~

## Step 3. Start Prometheus

1. Start the Prometheus server, with the `--config.file` flag pointing to the configuration file:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ prometheus --config.file=prometheus.yml
    ~~~

    ~~~
    ts=2022-04-20T18:57:45.857Z caller=main.go:516 level=info msg="Starting Prometheus" version="(version=2.34.0, branch=HEAD, revision=881111fec4332c33094a6fb2680c71fffc427275)"
    ts=2022-04-20T18:57:45.857Z caller=main.go:521 level=info build_context="(go=go1.17.8, user=root@d80b449ae319, date=20220315-15:04:36)"
    ...
    ts=2022-04-20T18:57:45.859Z caller=web.go:540 level=info component=web msg="Start listening for connections" address=localhost:9090
    ...
    ts=2022-04-20T18:57:46.811Z caller=main.go:1142 level=info msg="Loading configuration file" filename=prometheus.yml
    ts=2022-04-20T18:57:46.973Z caller=main.go:1179 level=info msg="Completed loading of configuration file" filename=prometheus.yml totalDuration=162.470967ms db_storage=1.057µs remote_storage=5.151µs web_handler=457ns query_engine=889ns scrape=150.34215ms scrape_sd=74.253µs notify=94.531µs notify_sd=26.921µs rules=11.488407ms tracing=19.58µs
    ts=2022-04-20T18:57:46.973Z caller=main.go:910 level=info msg="Server is ready to receive web requests."
    ~~~

2. Point your browser to `http://<hostname of machine running prometheus>:9090`, where you can use the Prometheus UI to query, aggregate, and graph CockroachDB time series metrics.
  - Prometheus auto-completes CockroachDB time series metrics for you, but if you want to see a full listing, with descriptions, point your browser to `http://<hostname of a CockroachDB node>:8080/_status/vars`.
  - For more details on using the Prometheus UI, see their [official documentation](https://prometheus.io/docs/introduction/getting_started/).

## Step 4. Send notifications with Alertmanager

Active monitoring helps you spot problems early, but it is also essential to send notifications when there are events that require investigation or intervention. In step 2, you already downloaded CockroachDB's starter [alerting rules](https://github.com/cockroachdb/cockroach/blob/master/monitoring/rules/alerts.rules.yml). Now, download, configure, and start [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/).

1. Download the [latest Alertmanager tarball](https://prometheus.io/download/#alertmanager) for your OS.

2. Extract the binary and add it to your system `PATH`. This makes it easy to start Alertmanager from any shell.

3. Make sure Alertmanager installed successfully:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ alertmanager --version
    ~~~

    ~~~
    alertmanager, version 0.24.0 (branch: HEAD, revision: f484b17fa3c583ed1b2c8bbcec20ba1db2aa5f11)
      build user:       root@8fd670bfea94
      build date:       20220325-09:24:35
      go version:       go1.17.8
    ~~~

4. [Edit the Alertmanager configuration file](https://prometheus.io/docs/alerting/configuration/) that came with the binary, `alertmanager.yml`, to specify the desired receivers for notifications.

5. Start the Alertmanager server, with the `--config.file` flag pointing to the configuration file:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ alertmanager --config.file=alertmanager.yml
    ~~~

6. Point your browser to `http://<hostname of machine running alertmanager>:9093`, where you can use the Alertmanager UI to define rules for [silencing alerts](https://prometheus.io/docs/alerting/alertmanager/#silences).

## Step 5. Visualize metrics in Grafana

Although Prometheus lets you graph metrics, [Grafana](https://grafana.com/) is a much more powerful visualization tool that integrates with Prometheus easily.

1. [Install and start Grafana for your OS](https://grafana.com/grafana/download).

2. Point your browser to `http://<hostname of machine running grafana>:3000` and log into the Grafana UI with the default username/password, `admin/admin`, or create your own account.

3. [Add Prometheus as a datasource](http://docs.grafana.org/datasources/prometheus/), and configure the datasource as follows:

    Field | Definition
    ------|-----------
    Name | Prometheus
    Default | True
    Type | Prometheus
    Url | `http://<hostname of machine running prometheus>:9090`
    Access | Direct

4. Download the starter [Grafana dashboards](https://github.com/cockroachdb/cockroach/tree/master/monitoring/grafana-dashboards) for CockroachDB:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/grafana-dashboards/runtime.json
    # runtime dashboard: node status, including uptime, memory, and cpu.
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/grafana-dashboards/storage.json
    # storage dashboard: storage availability.
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/grafana-dashboards/sql.json
    # sql dashboard: sql queries/transactions.
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ wget https://raw.githubusercontent.com/cockroachdb/cockroach/master/monitoring/grafana-dashboards/replicas.json
    # replicas dashboard: replica information and operations.
    ~~~

5. [Add the dashboards to Grafana](http://docs.grafana.org/reference/export_import/#importing-a-dashboard).

## See also

- [Monitoring and Alerting](monitoring-and-alerting.html)
