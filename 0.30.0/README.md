# What's new in Strimzi 0.30.0

## Prerequisites

1. Install Strimzi 0.30.0 using one of the available methods

## `StrimziPodSets`

2. Deploy a new Apache Kafka 3.2.0 cluster with 3 broker nodes and Cruise Control enabled.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.30.0/kafka.yaml
   ```

3. Wait for the Kafka cluster to get ready and check that:
     * There are no Kafka or ZooKeeper `StatefulSets` (`kubectl get statefulsets`).
     * The `StrimziPodSets` are used to manage the Kafka or ZooKeeper pods (`kubectl get strimzipodsets`).

## Restart Events

4. Edit the Kafka custom resource and disable the topic auto-creation.
   You can do that using `kubectl edit kafka my-cluster` and adding the following to the `.spec.kafka.config` section:
   ```yaml
   auto.create.topics.enable: "false"
   ```

5. Watch the Kubernetes events with `kubectl get events -w`.

6. When the operator rolls the Kafka brokers, you should see the following events in your log:
   ```
   $ kubectl get events -w
   LAST SEEN   TYPE      REASON                  OBJECT                                              MESSAGE
   ...
   0s          Normal    ConfigChangeRequiresRestart   pod/my-cluster-kafka-0                              Pod needs to be restarted, because reconfiguration cannot be done dynamically
   ...
   ```

### Cleanup

7. Once done you can delete all the Strimzi resources used during the demo:
   ```
   kubectl delete $(kubectl get strimzi -o name)
   ```
   And uninstall the operator.
