For kubernetes metrics collection, basically we will need to let promethues capature the following metrics:

By default the helm chart we installed will monitor its own targets: prometheus and alertmanager. Additional ones can be added setting a list with the [scrape_configs](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config) in the value `server.extraScrapeConfigs`. Here there is a simple example for wordpress (deployed in the default namespace):

For example:
```yaml
server:
  extraScrapeConfigs:
    - job_name: wordpress
      kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names:
            - default
      metrics_path: /metrics
      relabel_configs:
        - source_labels:
            - job
          target_label: __tmp_wordpress_job_name
        - action: keep
          source_labels:
            - __meta_kubernetes_service_label_app_kubernetes_io_instance
            - __meta_kubernetes_service_labelpresent_app_kubernetes_io_instance
          regex: (wordpress);true
        - action: keep
          source_labels:
            - __meta_kubernetes_service_label_app_kubernetes_io_name
            - __meta_kubernetes_service_labelpresent_app_kubernetes_io_name
          regex: (wordpress);true
        - action: keep
          source_labels:
            - __meta_kubernetes_endpoint_port_name
          regex: metrics
        - source_labels:
            - __meta_kubernetes_endpoint_address_target_kind
            - __meta_kubernetes_endpoint_address_target_name
          separator: ;
          regex: Node;(.*)
          replacement: ${1}
          target_label: node
        - source_labels:
            - __meta_kubernetes_endpoint_address_target_kind
            - __meta_kubernetes_endpoint_address_target_name
          separator: ;
          regex: Pod;(.*)
          replacement: ${1}
          target_label: pod
        - source_labels:
            - __meta_kubernetes_namespace
          target_label: namespace
        - source_labels:
            - __meta_kubernetes_service_name
          target_label: service
        - source_labels:
            - __meta_kubernetes_pod_name
          target_label: pod
        - source_labels:
            - __meta_kubernetes_pod_container_name
          target_label: container
        - action: drop
          source_labels:
            - __meta_kubernetes_pod_phase
          regex: (Failed|Succeeded)
        - source_labels:
            - __meta_kubernetes_service_name
          target_label: job
          replacement: ${1}
        - target_label: endpoint
          replacement: metrics
        - source_labels:
            - __address__
          target_label: __tmp_hash
          modulus: 1
          action: hashmod
        - source_labels:
            - __tmp_hash
          regex: 0
          action: keep
```

Below are four example scrape configurations that allow you to define targets and monitor corresponding metrics:

**Target 1: Ingress Endpoints

```
- job_name: 'ingress-endpoints'
  kubernetes_sd_configs:
    - role: endpoints
  follow_redirects: true
  enable_http2: true
  namespaces:
    names: [xxxxx]  # Replace 'xxxxx' with the namespace where your ingress controller is located
  relabel_configs:
    - source_labels: [__meta_kubernetes_service_name]
      separator: ';'
      regex: 'ingress-nginx-controller(-metrics|-defaultbackend)?'  # Update the regex according to your ingress controller service name
      target_label: service
      replacement: $1
      action: replace
    - source_labels: [__meta_kubernetes_endpoint_port_name]
      separator: ';'
      regex: 'metrics'
      replacement: $1
      action: keep
  # Include other necessary relabel rules as per your original configuration
  metric_relabel_configs:
    - source_labels: [__name__]
      separator: ';'
      regex: 'nginx_ingress_.*'
      replacement: $1
      action: keep

```

Then you will see the responding target definition and the following metrics explorer
![[Pasted image 20240528194829.png]]
***Target-2: kubernetes-nodes
```
- job_name: 'kubernetes-nodes'
  authorization:
    type: 'Bearer'
    credentials_file: '/var/run/secrets/kubernetes.io/serviceaccount/token'
  tls_config:
    ca_file: '/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
  kubernetes_sd_configs:
    - role: node
  relabel_configs:
    - source_labels: [__address__]
      regex: '(.+):10250'  # Assuming your nodes expose metrics on port 10250
      target_label: __address__
      replacement: $1
      action: keep
    - action: labelmap
      regex: '__meta_kubernetes_node_label_(.+)'
    - source_labels: [__meta_kubernetes_node_label_topology_kubernetes_io_region]
      regex: 'XXXXX'
      action: keep
    - source_labels: [__meta_kubernetes_node_label_node_kubernetes

```
Then you will see the responding target definition and the following metrics explorer
![[Pasted image 20240528200012.png]]

**Target-3: Cadvisor
**cAdvisor** is integrated into the kubelet on each node and provides container metrics. Prometheus can scrape cAdvisor directly
```
scrape_configs:
  - job_name: 'kubernetes-cadvisor'
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor


```
Then you will see the responding target definition and the following metrics explorer:
![[Pasted image 20240528195636.png]]
**Target-4: kube-state-metrics

If you need to collect Kubernetes state metrics (like pod and container statuses), kube-state-metrics is essential. It serves as the bridge between Kubernetes API data and Prometheus, which does not inherently understand Kubernetes object states without this additional service. Once kube-state-metrics is deployed and correctly configured in Prometheus, you should start seeing the desired metrics in your Prometheus instance

```
scrape_configs:
  - job_name: 'kube-state-metrics'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - XXXXX
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_name]
        action: keep
        regex: kube-state-metrics
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: http


```

Then you will see the responding target definition and the following metrics explorer:
![[Pasted image 20240528195825.png]]


**Reference:** For more details on Prometheus scrape configuration, visit the[scrape_configs](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)