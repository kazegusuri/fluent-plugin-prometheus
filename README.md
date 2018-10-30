# fluent-plugin-prometheus, a plugin for [Fluentd](https://www.fluentd.org)

[![Build Status](https://travis-ci.org/fluent/fluent-plugin-prometheus.svg?branch=master)](https://travis-ci.org/fluent/fluent-plugin-prometheus)

A fluent plugin that instruments metrics from records and exposes them via web interface. Intended to be used together with a [Prometheus server](https://github.com/prometheus/prometheus).

## Requirements

| fluent-plugin-prometheus | fluentd    | ruby   |
|--------------------------|------------|--------|
| 1.x.y                    | >= v0.14.8 | >= 2.1 |
| 0.x.y                    | >= v0.12.0 | >= 1.9 |

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'fluent-plugin-prometheus'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install fluent-plugin-prometheus

## Usage

fluentd-plugin-prometheus includes 6 plugins.

- `prometheus` input plugin
- `prometheus_monitor` input plugin
- `prometheus_output_monitor` input plugin
- `prometheus_tail_monitor` input plugin
- `prometheus` output plugin
- `prometheus` filter plugin

See [sample configuration](./misc/fluentd_sample.conf), or try [tutorial](#try-plugin-with-nginx).

### prometheus input plugin

You have to configure this plugin to expose metrics collected by other promtheus plugins.
This plugin provides a metrics HTTP endpoint to be scraped by a prometheus server on 24231/tcp(default).

With following configuration, you can access http://localhost:24231/metrics on a server where fluentd running.

```
<source>
  @type prometheus
</source>
```

More configuration parameters:

- `bind`: binding interface (default: '0.0.0.0')
- `port`: listen port (defaut: 24231)
- `metrics_path`: metrics HTTP endpoint (default: /metrics)

When using multiple workers, each worker binds to port + `fluent_worker_id`.

### prometheus_monitor input plugin

This plugin collects internal metrics in Fluentd. The metrics are similar to/part of [monitor_agent](http://docs.fluentd.org/articles/monitoring#monitoring-agent).

Current exposed metrics:

- `buffere_queue_length` of each BufferedOutput plugins
- `buffer_total_queued_size` of each BufferedOutput plugins
- `retry_count` of each BufferedOutput plugins

With following configuration, those metrics are collected.

```
<source>
  @type prometheus_monitor
</source>
```

More configuration parameters:

- `<labels>`: additional labels for this metric (optional). See [Labels](#labels)
- `interval`: interval to update monitor_agent information in seconds (default: 5)

### prometheus_output_monitor input plugin

**experimental**

This plugin collects internal metrics for output plugin in Fluentd. This is similar to `prometheus_monitor` plugin, but specialized for output plugin. There are Many metrics `prometheus_monitor` does not include, such as `num_errors`, `retry_wait` and so on.

Current exposed metrics:

- `fluentd_output_status_buffer_queue_length`
- `fluentd_output_status_buffer_total_bytes`
- `fluentd_output_status_retry_count`
- `fluentd_output_status_num_errors`
- `fluentd_output_status_emit_count`
- `fluentd_output_status_retry_wait`
    - current retry_wait computed from last retry time and next retry time
- `fluentd_output_status_emit_records`
    - only for v0.14
- `fluentd_output_status_write_count`
    - only for v0.14
- `fluentd_output_status_rollback_count`
    - only for v0.14


With following configuration, those metrics are collected.

```
<source>
  @type prometheus_output_monitor
</source>
```

More configuration parameters:

- `<labels>`: additional labels for this metric (optional). See [Labels](#labels)
- `interval`: interval to update monitor_agent information in seconds (default: 5)

### prometheus_tail_monitor input plugin

**experimental**

This plugin collects internal metrics for in_tail plugin in Fluentd. in_tail plugin holds internal state for files that the plugin is watching. The state is sometimes important to monitor plugins work correctly.

This plugin uses internal class of Fluentd, so it's easy to break.

Current exposed metrics:

- `fluentd_tail_file_position`
    - Current bytes which plugin reads from the file
- `fluentd_tail_file_inode`
    - inode of the file

Default labels:

- `plugin_id`: a value set for a plugin in configuration.
- `type`: plugin name. `in_tail` only for now.
- `path`: file path

With following configuration, those metrics are collected.

```
<source>
  @type prometheus_tail_monitor
</source>
```

More configuration parameters:

- `<labels>`: additional labels for this metric (optional). See [Labels](#labels)
- `interval`: interval to update monitor_agent information in seconds (default: 5)

### prometheus output/filter plugin

Both output/filter plugins instrument metrics from records. Both plugins have no impact against values of each records, just read.

Assuming you have following configuration and receiving message,

```
<match message>
  @type stdout
</match>
```

```
message {
  "foo": 100,
  "bar": 200,
  "baz": 300
}
```

In filter plugin style,

```
<filter message>
  @type prometheus
  <metric>
    name message_foo_counter
    type counter
    desc The total number of foo in message.
    key foo
  </metric>
</filter>

<match message>
  @type stdout
</match>
```

In output plugin style:

```
<filter message>
  @type prometheus
  <metric>
    name message_foo_counter
    type counter
    desc The total number of foo in message.
    key foo
  </metric>
</filter>

<match message>
  @type copy
  <store>
    @type prometheus
    <metric>
      name message_foo_counter
      type counter
      desc The total number of foo in message.
      key foo
    </metric>
  </store>
  <store>
    @type stdout
  </store>
</match>
```

With above configuration, the plugin collects a metric named `message_foo_counter` from key `foo` of each records.

See Supported Metric Type and Labels for more configuration parameters.

## Supported Metric Types

For details of each metric type, see [Prometheus documentation](http://prometheus.io/docs/concepts/metric_types/). Also see [metric name guide](http://prometheus.io/docs/practices/naming/).

### counter type

```
<metric>
  name message_foo_counter
  type counter
  desc The total number of foo in message.
  key foo
  label_key bar
  <labels>
    tag ${tag}
    host ${hostname}
    foo bar
  </labels>
</metric>
```

- `name`: metric name (required)
- `type`: metric type (required)
- `desc`: description of this metric (required)
- `key`: key name of record for instrumentation (**optional**)
- `label_key`: key name of record with label values (optional)
- `<labels>`: additional labels for this metric (optional). See [Labels](#labels)

If key is empty, the metric values is treated as 1, so the counter increments by 1 on each record regardless of contents of the record.

If label_key is specified, a label with that name will be added with the value from the record. If the value is an array,
a counter for each value is incremented. E.g. if you have `label_key exeception` and a record with `record['exception'] = ['MQException', 'InvalidDestinationException']`, you will get these metrics:

```
metric_name{exception="MQException"} 1.0
metric_name{exception="InvalidDestinationException"} 1.0
```

Other labels will be added as specified.

### gauge type

```
<metric>
  name message_foo_gauge
  type gauge
  desc The total number of foo in message.
  key foo
  <labels>
    tag ${tag}
    host ${hostname}
    foo bar
  </labels>
</metric>
```

- `name`: metric name (required)
- `type`: metric type (required)
- `desc`: description of metric (required)
- `key`: key name of record for instrumentation (required)
- `<labels>`: additional labels for this metric (optional). See [Labels](#labels)

### summary type

```
<metric>
  name message_foo
  type summary
  desc The summary of foo in message.
  key foo
  <labels>
    tag ${tag}
    host ${hostname}
    foo bar
  </labels>
</metric>
```

- `name`: metric name (required)
- `type`: metric type (required)
- `desc`: description of metric (required)
- `key`: key name of record for instrumentation (required)
- `<labels>`: additional labels for this metric (optional). See [Labels](#labels)

### histogram type

```
<metric>
  name message_foo
  type histogram
  desc The histogram of foo in message.
  key foo
  buckets 0.1, 1, 5, 10
  <labels>
    tag ${tag}
    host ${hostname}
    foo bar
  </labels>
</metric>
```

- `name`: metric name (required)
- `type`: metric type (required)
- `desc`: description of metric (required)
- `key`: key name of record for instrumentation (required)
- `buckets`: buckets of record for instrumentation (optional)
- `<labels>`: additional labels for this metric (optional). See [Labels](#labels)

## Labels

See [Prometheus Data Model](http://prometheus.io/docs/concepts/data_model/) first.

You can add labels with static value or dynamic value from records. In `prometheus_monitor` input plugin, you can't use label value from records.

### labels section

```
<labels>
  key1 value1
  key2 value2
</labels>
```

All labels sections has same format. Each lines have key/value for label.

You can use placeholder for label values. The placeholders will be expanded from records or reserved values. If you specify `${foo}`, it will be expanded by value of `foo` in record.

You can access nested fields in records via dot or bracket notation (https://docs.fluentd.org/v1.0/articles/api-plugin-helper-record_accessor#syntax), for example: `$.kubernetes.namespace`.

Reserved placeholders are:

- `${hostname}`: hostname
- `${worker_id}`: fluent worker id
- `${tag}`: tag name
  - only availabe in prometheus output/filter plugin


### top-level labels and labels inside metric

Prometheus output/filter plugin can have multiple metric section. Top-level labels section spcifies labels for all metrics. Labels section insede metric section specifis labels for the metric. Both are specified, labels are merged.

```
<filter message>
  @type prometheus
  <metric>
    name message_foo_counter
    type counter
    desc The total number of foo in message.
    key foo
    <labels>
      key foo
      data_type ${type}
    </labels>
  </metric>
  <metric>
    name message_bar_counter
    type counter
    desc The total number of bar in message.
    key bar
    <labels>
      key bar
    </labels>
  </metric>
  <labels>
    tag ${tag}
    hostname ${hostname}
  </labels>
</filter>
```

In this case, `message_foo_counter` has `tag`, `hostname`, `key` and `data_type` labels.


## Try plugin with nginx

Checkout repository and setup.

```
$ git clone git://github.com/fluent/fluent-plugin-prometheus
$ cd fluent-plugin-prometheus
$ bundle install --path vendor/bundle
```

Download pre-compiled prometheus binary and start it. It listens on 9090.

```
$ wget https://github.com/prometheus/prometheus/releases/download/v1.5.2/prometheus-1.5.2.linux-amd64.tar.gz -O - | tar zxf -
$ ./prometheus-1.5.2.linux-amd64/prometheus -config.file=./misc/prometheus.yaml -storage.local.path=./prometheus/metrics
```

Install Nginx for sample metrics. It listens on 80 and 9999.

```
$ sudo apt-get install -y nginx
$ sudo cp misc/nginx_proxy.conf /etc/nginx/sites-enabled/proxy
$ sudo chmod 777 /var/log/nginx && sudo chmod +r /var/log/nginx/*.log
$ sudo service nginx restart
```

Start fluentd with sample configuration. It listens on 24231.

```
$ bundle exec fluentd -c misc/fluentd_sample.conf -v
```

Generate some records by accessing nginx.

```
$ curl http://localhost/
$ curl http://localhost:9999/
```

Confirm that some metrics are exported via Fluentd.

```
$ curl http://localhost:24231/metrics
```

Then, make a graph on Prometheus UI. http://localhost:9090/

## Contributing

1. Fork it ( https://github.com/fluent/fluent-plugin-prometheus/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request


## Copyright

<table>
  <tr>
    <td>Author</td><td>Masahiro Sano <sabottenda@gmail.com></td>
  </tr>
  <tr>
    <td>Copyright</td><td>Copyright (c) 2015- Masahiro Sano</td>
  </tr>
  <tr>
    <td>License</td><td>Apache License, Version 2.0</td>
  </tr>
</table>
