For deploying Prometheus using Helm, I recommend using the open-source Bitnami charts available here: [Bitnami Charts for Grafana](https://github.com/bitnami/charts/tree/main/bitnami/grafana).

**Important Steps to Customize Your Grafana Deployment:**

1. **Set the Deployment Name:** Update the `fullnameOverride` in the `values.yaml` file to match your deployment specifics:
```
fullnameOverride: "grafana"
```
2. **Configure the Resources:** If you don't require the Alertmanager, set it to false. Modify the Prometheus configuration under the server section to specify resources and replica count:
```
resources:
  requests:
    cpu: 2
    memory: 512Mi
  limits:
    cpu: 4
    memory: 1024Mi

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



5. **Enable RBAC Settings:** According to the actual needs, specifies whether a ServiceAccount should be created and uutomount service account token for the application controller service account
```
serviceAccount:
create: true
name: ""
annotations: {}
automountServiceAccountToken: false
```


6. **Install Grafana Using Helm:** Execute the following command to install Prometheus with your customized `values.yaml`:
```
helm install my-release oci://registry-1.docker.io/bitnamicharts/grafana -f values.yaml
```

Nextly, I will continue introducing the custom configuration for Grafana User management(Oauth&Ldap integration) and Dashboard configuration.

**Reference:** For more details on Grafana configuration, see the [Parameters](https://github.com/bitnami/charts/tree/main/bitnami/grafana#parameters) section, More info at [Grafana documentation](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards).