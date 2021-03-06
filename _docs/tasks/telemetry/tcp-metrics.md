---
title: Collecting Metrics for TCP services

overview: This task shows you how to configure Istio to collect metrics for TCP services.

order: 25

layout: docs
type: markdown
---

{% include home.html %}

This task shows how to configure Istio to automatically gather telemetry for TCP
services in a mesh. At the end of this task, a new metric will be enabled for
calls to a TCP service within your mesh.

The [BookInfo]({{home}}/docs/guides/bookinfo.html) sample application is used
as the example application throughout this task.

## Before you begin
* [Install Istio]({{home}}/docs/setup/) in your cluster and deploy an
  application.

* This task assumes that the BookInfo sample will be deployed in the `default`
  namespace. If you use a different namespace, you will need to update the
  example configuration and commands.

* Install the Prometheus add-on. Prometheus
  will be used to verify task success. 
  ```bash
  kubectl apply -f install/kubernetes/addons/prometheus.yaml
  ```
  See [Prometheus](https://prometheus.io) for details.

## Collecting new telemetry data

1. Create a new YAML file to hold configuration for the new metrics that Istio
   will generate and collect automatically.

   Save the following as `tcp_telemetry.yaml`:

   ```yaml
    # Configuration for a metric measuring bytes sent from a server
    # to a client
    apiVersion: "config.istio.io/v1alpha2"
    kind: metric
    metadata:
    name: mongosentbytes
    namespace: default
    spec:
      value: connection.sent.bytes | 0 # uses a TCP-specific attribute
      dimensions:
        source_service: source.service | "unknown"
        source_version: source.labels["version"] | "unknown"
        destination_version: destination.labels["version"] | "unknown"
      monitoredResourceType: '"UNSPECIFIED"'
    ---
    # Configuration for a metric measuring bytes sent from a client
    # to a server
    apiVersion: "config.istio.io/v1alpha2"
    kind: metric
    metadata:
    name: mongoreceivedbytes
    namespace: default
    spec:
      value: connection.received.bytes | 0 # uses a TCP-specific attribute
      dimensions:
        source_service: source.service | "unknown"
        source_version: source.labels["version"] | "unknown"
        destination_version: destination.labels["version"] | "unknown"
      monitoredResourceType: '"UNSPECIFIED"'
    ---
    # Configuration for a Prometheus handler
    apiVersion: "config.istio.io/v1alpha2"
    kind: prometheus
    metadata:
    name: mongohandler
    namespace: default
    spec:
      metrics:
      - name: mongo_sent_bytes # Prometheus metric name
        instance_name: mongosentbytes.metric.default # Mixer instance name (fully-qualified)
        kind: COUNTER
        label_names:
        - source_service
        - source_version
        - destination_version
      - name: mongo_sent_bytes # Prometheus metric name
        instance_name: mongosentbytes.metric.default # Mixer instance name (fully-qualified)
        kind: COUNTER
        label_names:
        - source_service
        - source_version
        - destination_version
    ---
    # Rule to send metric instances to a Prometheus handler
    apiVersion: "config.istio.io/v1alpha2"
    kind: rule
    metadata:
    name: mongoprom
    namespace: default
    spec:
      match: context.protocol == "tcp"
             && destination.service = "mongodb.default.svc.cluster.local"
      actions:
      - handler: mongohandler.prometheus
        instances:
        - mongoreceivedbytes.metric
        - mongosentbytes.metric
   ```

1. Push the new configuration.

   ```bash
   istioctl create -f tcp_telemetry.yaml
   ```

   The expected output is similar to:
   ```
   Created config metric/default/mongosentbytes at revision 3852843
   Created config metric/default/mongoreceivedbytes at revision 3852844
   Created config prometheus/default/mongohandler at revision 3852845
   Created config rule/default/mongoprom at revision 3852846
   ```

1. Setup BookInfo to use MongoDB.

   1. Install `v2` of the `ratings` service:

      ```
      kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/kube/bookinfo-ratings-v2.yaml)
      ```

      Expected output:

      ```
      deployment "ratings-v2" configured
      ```

   1. Install the `mongodb` service:

      ```
      kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/kube/bookinfo-db.yaml)
      ```

      Expected output:

      ```
      service "mongodb" configured
      deployment "mongodb-v1" configured
      ```

   1. Add routing rules to send traffic to `v2` of the `ratings` service:

      ```
      kubectl apply -f samples/bookinfo/kube/route-rule-ratings-db.yaml
      ```

      Expected output:

      ```
      routerule "ratings-test-v2" created
      routerule "reviews-test-ratings-v2" created
      ```

1. Send traffic to the sample application.

   For the BookInfo sample, visit `http://$GATEWAY_URL/productpage` in your web
   browser or issue the following command:

   ```bash
   curl http://$GATEWAY_URL/productpage
   ```

1. Verify that the new metric values are being generated and collected.

   In a Kubernetes environment, setup port-forwarding for Prometheus by
   executing the following command:

   ```bash
   kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
   ```

   View values for the new metric via the [Prometheus UI](http://localhost:9090/graph#%5B%7B%22range_input%22%3A%221h%22%2C%22expr%22%3A%22mongo_received_bytes%22%2C%22tab%22%3A1%7D%5D).
   
   The provided link opens the Prometheus UI and executes a query for values of
   the `mongo_received_bytes` metric. The table displayed in the **Console** tab
   includes entries similar to:

   ```
   mongo_received_bytes{destination_version="v1",instance="istio-mixer.istio-system:42422",job="istio-mesh",source_service="ratings.default.svc.cluster.local",source_version="v2"}	2317
   ```

   NOTE: Istio also collects protocol-specific statistics for MongoDB. For
   example, the value of total OP_QUERY messages sent from the `ratings` service
   is collected in the following metric:
   `envoy_mongo_mongo_collection_ratings_query_total_counter` (click
   [here](http://localhost:9090/graph#%5B%7B%22range_input%22%3A%221h%22%2C%22expr%22%3A%22envoy_mongo_mongo_collection_ratings_query_total_counter%22%2C%22tab%22%3A1%7D%5D)
   to execute the query).

## Understanding TCP telemetry collection

In this task, you added Istio configuration that instructed Mixer to
automatically generate and report a new metric for all traffic to a TCP service
within the mesh.

Similar to the [Collecting Metrics and
Logs]({{home}}/docs/tasks/telemetry/metrics-logs.html) Task, the new
configuration consisted of _instances_, a _handler_, and a _rule_. Please see
that Task for a complete description of the components of metric collection.

Metrics collection for TCP services differs only in the limited set of
attributes that are available for use in _instances_. 

### TCP Attributes

Several TCP-specific attributes enable TCP policy and control within Istio.
These attributes are generated by server-side Envoy proxies and forwarded to
Mixer at both connection establishment and connection close. Additionally,
context attributes provide the ability to distinguish between `http` and `tcp`
protocols within policies.

<figure><img style="max-width:100%;" src="./img/istio-tcp-attribute-flow.svg" alt="Attribute Generation Flow for TCP Services in an Istio Mesh." title="TCP Attribute Flow" />
<figcaption>TCP Attribute Flow</figcaption></figure>

## Cleanup

* Remove the new telemetry configuration:

  ```bash
  istioctl delete -f tcp_telemetry.yaml
  ```

* If you are not planning to explore any follow-on tasks, refer to the
  [BookInfo cleanup]({{home}}/docs/guides/bookinfo.html#cleanup) instructions
  to shutdown the application.

## Further reading

* Learn more about [Mixer]({{home}}/docs/concepts/policy-and-control/mixer.html)
  and [Mixer
  Config]({{home}}/docs/concepts/policy-and-control/mixer-config.html).

* Discover the full [Attribute
  Vocabulary]({{home}}/docs/reference/config/mixer/attribute-vocabulary.html).

* Read the reference guide to [Writing
  Config]({{home}}/docs/reference/writing-config.html).

* Refer to the [In-Depth Telemetry]({{home}}/docs/guides/telemetry.html) guide.

* Learn more about [Querying Istio
   Metrics]({{home}}/docs/tasks/telemetry/querying-metrics.html).

* Learn more about the [MongoDB-specific statistics generated by
  Envoy](https://envoyproxy.github.io/envoy/configuration/network_filters/mongo_proxy_filter.html#statistics).

