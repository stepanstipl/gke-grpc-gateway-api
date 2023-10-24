# Streaming gRPC on GKE with Gateway API

Steps to deploy gRPC server streaming service running on GKE behind Google's
Global External Application Load Balancer using the Gateway API, Google-managed
certificate and Envoy proxy for TLS termination at the backends.

## Prerequisites

These steps expect GKE cluster with a Gateway Controller [^1] and internet
access (to download the prebuilt container image). Also the usual `gcloud` CLI
configured for your project and `kubectl` with credentials to your cluster.
You'll also need working DNS subdomain to point to the load balancer IP.

## Setup Steps

- Create static IP
  ```shell
  $ gcloud compute addresses create grpcdemo-vip \
    --ip-version=IPV4 \
    --global
  ```

- Point the public DNS to the previously created global IP (I'll be
  using `grpcdemo.example.com`)

- Create Google-managed SSL certificate
  ```shell
  $ gcloud compute ssl-certificates create grpcdemo-cert \
    --domains=grpcdemo.example.com \
    --global
  ```

- Generate self-signed certificate for the backend
  ```shell
  $ openssl ecparam -genkey -name prime256v1 -noout -out key.pem
  $ openssl req -x509 -new -key key.pem -out cert.pem -days 3650 -subj '/CN=internal'
  ```
  TLS is required both between client and GFE, as well as GFE and backend [^2].

  **Important:** The certificate has to use one of supported signatures
  compatible with BoringSSL, see [^3][^4] for more details. 

- Create K8S Secret with the self-signed cert
  ```shell
  $ kubectl create secret tls grpcdemo-tls \
  --cert=cert.pem \
  --key=key.pem
  ```

- Create K8S Configmap with Envoy config
  ```shell
  $ kubectl create configmap envoy-config --from-file=envoy.yaml
  ```

- Deploy _grpcdemo_ app
  ```shell
  $ kubectl apply -f manifests/deploy-grpcdemo.yaml
  ```

  We're using the *Cloud Run gRPC Server Streaming sample application*[^5] which listens on port 8080.
  The Envoy then listens on port 8443 with self-signed certificate.

- Deploy _grpcdemo_ svc
  ```shell
  $ kubectl apply -f manifests/svc-grpcdemo.yaml
  ```

- Deploy _grpcdemo_ *gke-l7-gxlb* gateway
  ```shell
  $ kubectl apply -f manifests/gateway-grpcdemo.yaml
  ```

- Deploy _grpcdemo_ HealthCheckPolicy
  ```shell
  $ kubectl apply -f manifests/httproute-health.yaml
  ```

- Deploy _grpcdemo_ HTTPRoute
  ```shell
  $ kubectl apply -f manifests/httproute-grpcdemo.yaml
  ```
  *Don't forget to change the DNS in the manifest.*

- Test

  Clone the repository [^5] and build the client:
  ```shell
  $ git clone https://github.com/GoogleCloudPlatform/golang-samples
  $ cd golang-samples/run/grpc-server-streaming
  $ go build -o cli ./client
  ```

  And run the client:
  ```shell
  $ ./cli -server grpcdemo.example.com:443
  rpc established to timeserver, starting to stream
  received message: current_timestamp: 2023-10-24T19:15:32Z
  received message: current_timestamp: 2023-10-24T19:15:33Z
  received message: current_timestamp: 2023-10-24T19:15:34Z
  ...
  ```


[^1]: https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api
[^2]: https://cloud.google.com/load-balancing/docs/https#using_grpc_with_your_applications
[^3]: https://github.com/grpc/grpc/issues/6722
[^4]: https://groups.google.com/a/chromium.org/forum/#!msg/blink-dev/kWwLfeIQIBM/9chGZ40TCQAJ
[^5]: https://github.com/GoogleCloudPlatform/golang-samples/tree/main/run/grpc-server-streaming
