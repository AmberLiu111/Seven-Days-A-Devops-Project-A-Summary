Ingress controllers play a critical role in managing access to services within a Kubernetes environment. They act as a gateway for external traffic, routing requests to the appropriate services based on the rules defined in Ingress resources. I want to use this article to provides a basic comparison of popular ingress controller proxy base—NGINX, HAProxy, Envoy, and introduces configuration insights and examples for Ingress-nginx, HAProxy Ingress and Contour Ingress controllers.

## 1. Ingress controller comparing:

Comparative Analysis of Leading Ingress Controllers can be found here:
https://docs.google.com/spreadsheets/d/191WWNpjJ2za6-nbG4ZoUMXMpUK8KlCIosvQB0f-oq3k/view?gid=907731238#gid=907731238

Comparison between NGINX, HAProxy, and Envoy as ingress proxy base.
### NGINX

NGINX excels in advanced traffic management and security, facilitating A/B testing through weight-based distribution of incoming requests and enhancing content delivery with efficient caching mechanisms at the network edge. Its security capabilities are bolstered by the integration of ModSecurity in NGINX Plus, which allows for extensive monitoring and control of HTTP traffic. Additionally, rate limiting features ensure consistent service performance during peak loads. NGINX is widely used in high-traffic websites and as a reverse proxy, providing robust scalability and security in diverse environments.

### HAProxy

HAProxy is renowned for its precise load balancing using multiple algorithms like round-robin and least connections, along with effective session persistence necessary for applications requiring continuous user sessions. It ensures high availability through sophisticated health checks and seamless failover to backup servers, making it ideal for mission-critical applications that demand high reliability. HAProxy is commonly employed in financial services and e-commerce platforms, where maintaining uninterrupted service is critical.

### Envoy

Envoy supports modern application protocols such as HTTP/2 and gRPC, making it highly suitable for microservices architectures. It dynamically adjusts routing and service discovery as network components change, with robust observability features including detailed metrics and integration with distributed tracing tools like Zipkin and Jaeger. Envoy shines in real-world deployments as a sidecar proxy in service meshes, handling east-west traffic within clusters to enhance security, resilience, and inter-service communication. It is particularly favored in industries adopting cloud-native technologies, helping streamline operations across distributed systems.

## 2. In-Depth Look at NGINX Ingress Controller

### (1)Ingress-nginx Ingress frequent used annotations:
[nginx.ingress.kubernetes.io/auth-url](http://nginx.ingress.kubernetes.io/auth-url)  
[nginx.ingress.kubernetes.io/backend-protocol](http://nginx.ingress.kubernetes.io/backend-protocol)      
[nginx.ingress.kubernetes.io/client_header_buffer_size](http://nginx.ingress.kubernetes.io/client_header_buffer_size)      
[nginx.ingress.kubernetes.io/configuration-snippet](http://nginx.ingress.kubernetes.io/configuration-snippet)  
[nginx.ingress.kubernetes.io/cors-allow-credentials](http://nginx.ingress.kubernetes.io/cors-allow-credentials)  
[nginx.ingress.kubernetes.io/cors-allow-headers](http://nginx.ingress.kubernetes.io/cors-allow-headers)  
[nginx.ingress.kubernetes.io/cors-allow-methods](http://nginx.ingress.kubernetes.io/cors-allow-methods)  
[nginx.ingress.kubernetes.io/cors-allow-origin](http://nginx.ingress.kubernetes.io/cors-allow-origin)  
[nginx.ingress.kubernetes.io/enable-cors](http://nginx.ingress.kubernetes.io/enable-cors)  
[nginx.ingress.kubernetes.io/large-client-header-buffers](http://nginx.ingress.kubernetes.io/large-client-header-buffers)      
[nginx.ingress.kubernetes.io/limit-rps](http://nginx.ingress.kubernetes.io/limit-rps)      
[nginx.ingress.kubernetes.io/proxy-body-size](http://nginx.ingress.kubernetes.io/proxy-body-size)      
[nginx.ingress.kubernetes.io/proxy-buffer-size](http://nginx.ingress.kubernetes.io/proxy-buffer-size)      
[nginx.ingress.kubernetes.io/proxy-buffers-number](http://nginx.ingress.kubernetes.io/proxy-buffers-number)      
[nginx.ingress.kubernetes.io/proxy-connect-timeout](http://nginx.ingress.kubernetes.io/proxy-connect-timeout)      
[nginx.ingress.kubernetes.io/proxy-read-timeout](http://nginx.ingress.kubernetes.io/proxy-read-timeout)      
[nginx.ingress.kubernetes.io/proxy-send-timeout](http://nginx.ingress.kubernetes.io/proxy-send-timeout)      
[nginx.ingress.kubernetes.io/server-snippet](http://nginx.ingress.kubernetes.io/server-snippet)  
[nginx.ingress.kubernetes.io/upstream-vhost](http://nginx.ingress.kubernetes.io/upstream-vhost)

You can add different additional annotations to your ingresses that apply and change the way how that ingress works, these are documented at [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)
### (2)Ingress-nginx ingress https to http redirect
If the `tls:` section is not set, NGINX will provide the default certificate but will not force HTTPS redirect.

On the other hand, if the `tls:` section is set - even without specifying a `secretName` option - NGINX will force HTTPS redirect.

To force redirects for Ingresses that do not specify a TLS-block at all, take a look at `force-ssl-redirect` in [ConfigMap](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/).
https://kubernetes.github.io/ingress-nginx/user-guide/tls/

### (3)Ingress-nginx contoller default tls/ssl certificate
The Ingress controller provides the flag `--default-ssl-certificate`. The secret referred to by this flag contains the default certificate to be used when accessing the catch-all server. If this flag is not provided NGINX will use a self-signed certificate.

For instance, if you have a TLS secret `foo-tls` in the `default` namespace, add `--default-ssl-certificate=default/foo-tls` in the `nginx-controller` deployment.

You can generate a self-signed certificate and private key with:
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ${KEY_FILE} -out ${CERT_FILE} -subj "/CN=${HOST}/O=${HOST}" -addext "subjectAltName = DNS:${HOST}"`
```

Then create the secret in the cluster via:
```
`kubectl create secret tls ${CERT_NAME} --key ${KEY_FILE} --cert ${CERT_FILE}`
```

The resulting secret will be of type `kubernetes.io/tls`

**Note**: Ensure that the certificate order is leaf->intermediate->root, otherwise the controller will not be able to import the certificate, and you'll see this error in the logs `W1012 09:15:45.920000 6 backend_ssl.go:46] Error obtaining X.509 certificate: unexpected error creating SSL Cert: certificate and private key does not have a matching public key: tls: private key does not match public key`

### (4)Ingress-nginx annotation ssl-passthrough: “true”

This feature is implemented by intercepting **all traffic** on the configured HTTPS port (default: 443) and handing it over to a local TCP proxy. This bypasses NGINX completely and introduces a non-negligible performance penalty.

SSL Passthrough works on layer 4 of the OSI model (TCP) and not on layer 7 (HTTP).

Backend service should work on HTTPS and have a valid SSL as it will be used for SSL/TLS negotiation with the client.

**When you need that option set to “true”**

- You want to make SSL termination on your backend service because you have a personal SSL certificate installed in your backend service. You do not want to use that SSL on the nginx ingress
- For some reason, you want more security by bypassing nginx ingress server and decrypting requests on the backend service
- You have a backend service that is responsible for handling access logs and metrics and you want to leverage from that capabilities

**What are the consequences from that option set to “true”**

- You cannot use SSL kubernetes secret, assign it to ingress, and expect to use it for SSL/TLS communication with the client
- Because SSL Passthrough works on layer 4 of the OSI model (TCP) and not on layer 7 (HTTP), using SSL Passthrough invalidates all the other annotations set on an Ingress object.
- You will not have access logs and nginx ingress metrics as requests are encrypted and they are just piped to backend service

### (5)Ingress-nginx controller integrate with aws cert manager
You can configure TLS support via the following annotations:

- `service.beta.kubernetes.io/aws-load-balancer-ssl-cert` specifies the ARN of one or more certificates managed by the [AWS Certificate Manager](https://aws.amazon.com/certificate-manager).
    
    The first certificate in the list is the default certificate and remaining certificates are for the optional certificate list. See [Server Certificates](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/create-tls-listener.html#tls-listener-certificates) for further details.
    `service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-west-2:xxxxx:certificate/xxxxxxx`
    
(6)Ingress-nginx controller integrate with google certificate manager
 But cannot use the cert from google cert manager directly. cert needs to be imported as secret.
 
However there is no annotation to let nginx ingress controller integrate google certificate manager
If you add annotation `networking.gke.io/managed-certificates: "managed-certificate"` to Ingress class `nginx`, the site will be insecure and report "Kubernetes Ingress Controller Fake Certificate". That annotation only works for Ingress class `gce`. The [prerequisites](https://cloud.google.com/kubernetes-engine/docs/how-to/managed-certs#prerequisites) say your "kubernetes.io/ingress.class" must be "gce".

For nginx, certificates must be added as a Kubernetes secret. This will lose the self-rotating feature of the google managed certificates that only work with Ingress of class `gce`.

same as ingress.gcp.kubernetes.io/pre-shared-cert annotation.

# 3.Contour ingress controller introduction and examples

Contour, developed by VMware, is an advanced Layer 7 ingress controller that leverages Envoy as its data plane. Positioned behind a Network Load Balancer in AWS, Contour is designed to enhance routing functionality significantly compared to NGINX, boasting a 12-15% improvement in Requests Per Second during initial performance tests. For comprehensive settings and in-depth documentation, please visit the [Contour Documentation](https://projectcontour.io/docs) and its [GitHub Repository](https://github.com/projectcontour/contour). Envoy's documentation and repository can be found [here](https://www.envoyproxy.io/docs/envoy/v1.21.0/) and [here](https://github.com/envoyproxy/envoy) respectively.

### **(1)Ingress vs HTTPProxy**

Contour supports both traditional Kubernetes Ingress and the more advanced HTTPProxy API. HTTPProxy offers a richer and more robust configuration landscape, natively supporting service weighting and load balancing strategies. It's particularly advantageous in multi-team Kubernetes clusters due to its ability to delegate routing configurations across different namespaces. Unlike Ingress, HTTPProxy validates configurations at creation, supporting complex multi-service routes that enhance fault tolerance and traffic managemen

| Ingress                                  | IngressRoute --> HTTPProxy                                                             |
| ---------------------------------------- | -------------------------------------------------------------------------------------- |
| Uses annotations                         | Natively supports service weighting and load balancing strategy                        |
| Does not support multi-team K8s clusters | Enables delegation of routing configuration for path + header or domain to another NS  |
|                                          | Supports multi-team K8s clusters                                                       |
|                                          | Validates HTTPProxy objects at creation time                                           |
|                                          | Accepts multiple services within a single route and load balances traffic across them. |


### **(2)Configuration Examples: Realizing Contour's Full Potential**
#### **Basic HTTPProxy Configuration**

This example demonstrates a basic HTTPProxy configuration managing traffic across multiple services, showcasing Contour’s capability to handle precise routing conditions and service-specific requirements:
```
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: root
  namespace: projectcontour
spec:
  virtualhost:
    fqdn: contour.demo.test
  routes:
    - services:
      - name: site
        port: 80
      conditions:
      - prefix: /secure
      - header:
          name: User-Agent
          contains: Chrome

```
#### **Cross namespaces example:**
The following configuration exemplifies Contour’s cross-namespace capabilities, allowing for shared routing configurations that benefit large, distributed environments:
```
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: root
  namespace: projectcontour
spec:
  virtualhost:
    fqdn: contour.demo.test
  includes:
    - name: blog
      namespace: projectcontour-blog
      conditions:
        - prefix: /blog
  routes:
    - services:
      - name: site
        port: 80
      conditions:
      - prefix: /secure
      - header:
          name: User-Agent
          contains: Chrome

```

#### **Timeouts and Retries**
```
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: timeout-retry-example
  namespace: projectcontour
spec:
  virtualhost:
    fqdn: app.example.com
    tls: secretName: my-tls-cert
  routes:
    - conditions:
      - prefix: /app
      services:
        - name: app-service
          port: 80
          responseTimeout: 30s  # Wait up to 30 seconds for a response
          retryPolicy:
            count: 5             # Retry failed requests up to 5 times
            perTryTimeout: 5s    # Wait up to 5 seconds per retry
            retryOn: "5xx"       # Retry on 5xx status codes

```
#### **Load Balancing Strategy**
Contour allows specifying different load balancing strategies, such as Round Robin or Weighted Least Request. This example shows how to implement a weighted load balancing strategy.

```
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: load-balancing-example
  namespace: projectcontour
spec:
  virtualhost:
    fqdn: lb.example.com
  routes:
    - conditions:
      - prefix: /app
      services:
        - name: app-service1
          port: 80
          weight: 80  # 80% of traffic goes here
        - name: app-service2
          port: 80
          weight: 20  # 20% of traffic goes here

```

# 4.HAProxy ingress controller introduction and examples
HAProxy, renowned for its high performance and reliability, serves as a robust ingress controller in Kubernetes environments. As an ingress controller, HAProxy uses its proven load balancing technology to efficiently distribute traffic across your Kubernetes services. Positioned to compete with other popular solutions like NGINX and Contour, HAProxy distinguishes itself with its advanced traffic management capabilities and its focus on high availability and security.

For more detailed information on HAProxy's features and configurations, visit the HAProxy Documentation and its [GitHub Repository](https://github.com/haproxy/haproxy).

## **(1) Features and Advantages**

HAProxy excels in areas including SSL termination, real-time traffic monitoring, and advanced load balancing algorithms. It supports a wide range of traffic routing and manipulation techniques, making it an ideal choice for complex application landscapes.

### Comparison Table: HAProxy vs. Other Ingress Controllers

|Feature|HAProxy Ingress|Other Ingress Controllers|
|---|---|---|
|SSL Termination|Supported natively|Supported by most, but with variations|
|Real-time Monitoring|Extensive support with HAProxy Stats|Basic support, varies by implementation|
|Load Balancing Algorithms|Multiple algorithms like round-robin, least connections|Often limited to fewer choices|
|High Availability|Built-in high availability features|Depends on external configuration|
|Advanced Traffic Routing|Advanced ACLs and content-based routing|Basic routing capabilities|

## **(2) Configuration Examples: Leveraging HAProxy's Capabilities**

### **Basic HAProxy Configuration**

This example demonstrates how HAProxy can be used to manage ingress traffic effectively by routing based on hostname and path:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: haproxy-ingress-example
  namespace: default
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80

```

### **Using ACLs for Advanced Routing**

HAProxy's powerful ACL capabilities allow for more granular traffic routing. Here’s an example of routing based on a specific header:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: haproxy-acl-routing
  namespace: default
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
  ingressClassName: haproxy

```
### **SSL Termination and HTTPS Redirection**

This configuration sets up HAProxy to handle SSL termination and redirect HTTP traffic to HTTPS:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: haproxy-ssl-termination
  namespace: default
annotations:
  haproxy.org/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 443
```

### **Load Balancing Strategies**

Implement different load balancing strategies such as round-robin or least connections to optimize service performance:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: haproxy-load-balancing
  namespace: default
annotations:
  haproxy.org/load-balance: "leastconn"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: balanced-service
            port:
              number: 80
```

**Note**: HAProxy Ingress v0.14 partially supports the Gateway API spec, v1alpha1 and v1alpha2 versions.
## References

- Contour
    - [Documentation](https://projectcontour.io/docs/v1.20.0/)
    - [Repository](https://github.com/projectcontour/contour)
- Envoy
    - [Documentation](https://www.envoyproxy.io/docs/envoy/v1.21.0/)
    - [Repository](https://github.com/envoyproxy/envoy)
- Kubernetes Gateway API
    - [API](https://github.com/kubernetes-sigs/gateway-api)
    - [Documentation](https://gateway-api.sigs.k8s.io/)
- Kubernetes Ingress API  
    - [API](https://pkg.go.dev/k8s.io/api/networking/v1beta1)
    - [Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- HAProxy
    - [Ingress Controller supports Gateway API](https://haproxy-ingress.github.io/docs/configuration/gateway-api/)