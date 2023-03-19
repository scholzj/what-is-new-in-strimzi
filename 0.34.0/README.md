# What's new in Strimzi 0.34.0

## Prerequisites

1. Install Strimzi 0.34.0 using one of the available methods.

## Preparation

2. Deploy a new Apache Kafka cluster and wait until it is ready.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.34.0/kafka.yaml
   ```

## Deploy Kafka Connect with Connectors

3. Create a new topic `my-topic`.
   You can use the [`topic.yaml`](./topic.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.34.0/topic.yaml
   ```

4. Deploy a new Connect cluster and wait until it is ready.
   Use the [`connect.yaml`](./connect.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.34.0/connect.yaml
   ```
   The Connect deployment adds the [Echo Sink connector](https://github.com/scholzj/echo-sink) and the [Camel Timer connector](https://mvnrepository.com/artifact/org.apache.camel.kafkaconnector/camel-timer-kafka-connector/0.11.5) to be able to send and receive messages.

5. Check the names of the Kafka Connect pods created by the deployment.
   Notice their _random_ names such as `my-connect-connect-85c5598cf7-mqrgr`.
   You can also check the Kubernetes Deployment managing the Kafka Connect cluster:
   ```
   $ kubectl get deployment my-connect-connect
   NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
   my-connect-connect   3/2     3            3           13m
   ```

6. Create the connectors.
   You can use the [`connectors.yaml`](./connectors.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.34.0/connectors.yaml
   ```
   The connectors and their tasks should start and send and receive messages.
   You can check the logs of the Kafka Connect pods to see that the messages are being sent and received

## Enable Stable Pod-identities

7. To use the table pod identities in Kafka Connect, you have to enable the `StableConnectIdentities` feature gate
   You can do it by setting the `STRIMZI_FEATURE_GATES` environment variable in the Cluster Operator to contain `+StableConnectIdentities`.
   You can do that by editing the deployment, editing the Operator Hub subscription, or using `kubectl set env` command.
   ```
   kubectl set env deployment/strimzi-cluster-operator STRIMZI_FEATURE_GATES=+StableConnectIdentities
   ```

8. Watch the Kafka connect pods roll one by one.
   The new pods will have stable names which will not change with every restart such as `my-connect-connect-0` or `my-connect-connect-1`.
   They will be also managed by a `StrimziPodSet` resource instead of Kubernetes Deployment:
   ```
   $ kubectl get strimzipodset my-connect-connect
   NAME                 PODS   READY PODS   CURRENT PODS   AGE
   my-connect-connect   3      3            3              4m15s
   ```
   You can check the logs from the Connect pods to see that the connectors still work.

## On your own

9. The `StableConnectIdentities` feature gate applies also to Kafka Mirror Maker 2 clusters.
   You can try it with Mirror Maker 2 as well.

## Cleanup

10. Once done you can delete all the Strimzi resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
