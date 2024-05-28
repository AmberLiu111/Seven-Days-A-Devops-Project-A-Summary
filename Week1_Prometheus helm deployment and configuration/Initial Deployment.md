For deploying Prometheus using Helm, I recommend using the open-source Bitnami charts available here: [Bitnami Charts for Prometheus](https://github.com/bitnami/charts/tree/main/bitnami/prometheus).

**Important Steps to Customize Your Prometheus Deployment:**

1. **Set the Deployment Name:** Update the `fullnameOverride` in the `values.yaml` file to match your deployment specifics:
```
fullnameOverride: "prometheus"
```
2. **Configure the Alertmanager:** If you don't require the Alertmanager, set it to false. Modify the Prometheus configuration under the server section to specify resources and replica count:
```
resources:
  requests:
    cpu: 4
    memory: 10Gi
  limits:
    cpu: 6
    memory: 14Gi

replicaCount: 1
```
    Note: It's crucial to experiment with different resource settings to avoid Out-of-Memory (OOM) errors during pod initialization.
    
3. **Setup Ingress for Dashboard Access:** Define the ingress host to access the Prometheus dashboard without needing port forwarding:

```
ingress:
  enabled: true
  pathType: ImplementationSpecific
  hostname: xxxxx.com
```



4. **Define Storage Options:** Specify the storage class and size under the persistence section:

```
persistence:
  storageClass: xxxx
  annotations: {}
  accessModes:
    - ReadWriteMany  # or ReadWriteOnly
  size: xxxx

```



5. **Enable RBAC Settings:** Configure RBAC to ensure that Prometheus can scrape metrics across different namespaces. This configuration will automatically create the necessary `ClusterRole`, `ClusterRoleBinding`, and `ServiceAccount` based on your specified rules. This step is crucial for enabling comprehensive metrics collection throughout your Kubernetes environment.
```
rbac:
  create: true
  rules:
    - apiGroups:
        - "*"
      resources:
        - "*"
      verbs:
        - "*"

```


6. **Install Prometheus Using Helm:** Execute the following command to install Prometheus with your customized `values.yaml`:
```
helm install my-release oci://registry-1.docker.io/bitnamicharts/prometheus -f values.yaml
```

**Reference:** For more details on Prometheus configuration, visit the [official Prometheus website](https://prometheus.io/).