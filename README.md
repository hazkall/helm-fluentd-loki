# Fluentd Helm Chart with Grafana Loki Plugin

This chart install fluentd daemonset pre-configured to send logs to Grafana Loki.

<https://grafana.com/docs/loki/latest/clients/fluentd/>

<https://grafana.com/oss/loki/>

The default configurations bellow are required on values.yaml for the fluentd pod to be able connect to the loki server.

```yaml
env:
- name: "FLUENTD_CONF"
  value: "fluent.conf"
- name: "LOKI_URL"
  value: "http://loki-server:3100"
- name: "LOKI_USERNAME"
  value: ""
- name: "LOKI_PASSWORD"
  value: ""
```

## Install Fluentd-Loki

```bash

helm repo add fluentd-loki https://ativy-digital.github.io/helm-fluentd-loki/

helm repo update

# Logging is a sugested name for namespace, you can user any namespace or not define and use default.
helm install RELEASE fluentd-loki/fluentd-loki -n logging

```

## Configuration of fluentd

Below is the configuration of fluentd, to collect and parse kubernetes logs to the loki server. This can be changed on values.yaml

```yaml

fileConfigs:
  fluent.conf: |-
    ################################################################
    # This source gets all logs from local container runtime host
    @include pods-fluent.conf
    @include prometheus.conf
    @include loki-fluent.conf

  pods-fluent.conf: |-
    <source>
      @type tail
      @id in_tail_container_logs
      read_from_head true
      tag kubernetes.*
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      exclude_path ["/var/log/containers/fluent*"]
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key time
          time_type string
          time_format "%Y-%m-%dT%H:%M:%S.%NZ"
          keep_time_key false
        </pattern>
        <pattern>
          format regexp
          expression /^(?<time>.+) (?<stream>stdout|stderr)( (.))? (?<log>.*)$/
          time_format '%Y-%m-%dT%H:%M:%S.%NZ'
          keep_time_key false
        </pattern>
      </parse>
      emit_unmatched_lines true
    </source>

    <match fluentd.**>
      @type null
    </match>

    <match kubernetes.var.log.containers.**fluentd**.log>
      @type null
    </match>

    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "#{'https://' + ENV.fetch('KUBERNETES_SERVICE_HOST') + ':' + ENV.fetch('KUBERNETES_SERVICE_PORT') + '/api'}"
      verify_ssl "true"
      ca_file "#{ENV['KUBERNETES_CA_FILE']}"
    </filter>

    <filter kubernetes.var.log.containers.**>
      @type record_transformer
      enable_ruby
      remove_keys kubernetes, docker

      <record>
        #app ${ record.dig("kubernetes", "labels", "app") }
        namespace ${ record.dig("kubernetes", "namespace_name") }
        pod ${ record.dig("kubernetes", "pod_name") }
        container ${ record.dig("kubernetes", "container_name") }
      </record>

    </filter>

  prometheus.conf: |-
    <source>
      @type prometheus
      @id in_prometheus
      bind "0.0.0.0"
      port 24231
      metrics_path "/metrics"
    </source>

  loki-fluent.conf: |-
    <match kubernetes.**>
      @type loki
      url "#{ENV['LOKI_URL']}"
      username "#{ENV['LOKI_USERNAME']}"
      password "#{ENV['LOKI_PASSWORD']}"
      extract_kubernetes_labels true
      label_keys "namespace,pod,container"
      extra_labels {"collector":"fluentd"}
      flush_interval 10s
      flush_at_shutdown true
      buffer_chunk_limit 1m
      line_format key_value
    </match>


```

## Helm Chart Maintainers

@lipenodias (github)

@victorfbraga (github)
