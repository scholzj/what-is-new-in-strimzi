# What's new in Strimzi 0.29.0

## Prerequisites

1. Install Strimzi 0.29.0 using one of the available methods

## New Cruise Control rebalance modes

2. Deploy a new Apache Kafka 3.2.0 cluster with 3 broker nodes and Cruise Control enabled.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.29.0/kafka.yaml
   ```

3. Wait for the Kafka cluster to get ready.
   And once it is ready, create a new topic named `my-topic` with replication factor `3` and `10` replicas.
   You can use the [`topic.yaml` file](./topic.yaml) for it.
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.29.0/topic.yaml
   ```
   Once the topic is created, describe all the topics and check how they are all scheduled to the brokers 0, 1 and 2.
   ```
   kubectl run kafka-clients -ti --image=quay.io/strimzi/kafka:0.29.0-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe
   ```

4. Scale the Kafka cluster to 5 nodes.
   Edit the Kafka CR using `kubectl edit kafka my-cluster` and set `.spec.kafka.replicas` to `5`.
   Wait until the new brokers start.
   Check the topics again and you should see that nothing changed.
   All partitions are on brokers 0, 1 and 2 and the brokers 4 and 5 are empty.
   ```
   kubectl run kafka-clients -ti --image=quay.io/strimzi/kafka:0.29.0-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe
   ```

5. Trigger the rebalance to _add_ the brokers to the cluster by scheduling some topics to them.
   Use the [`add-brokers.file` file](./add-brokers.yaml) file to do it.
   Notice the `.spec.mode` and `.spec.brokers` sections of the KafkaRebalance resource.
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.29.0/add-brokers.yaml
   ```
   Wait for the proposal to become ready:
   ```
   kubectl get kafkarebalance add-brokers -o wide -w
   ```
   And then approve it:
   ```
   kubectl annotate kafkarebalance add-brokers strimzi.io/rebalance=approve
   ```
   And wait for the rebalance to complete:
   ```
   kubectl get kafkarebalance add-brokers -o wide -w
   ```
   Once it is complete, check the topics again.
   You should see that some partitions now have replicas on the brokers 3 and 4.
   But no partitions should move between the nodes 0, 1 and 2.
   ```
   kubectl run kafka-clients -ti --image=quay.io/strimzi/kafka:0.29.0-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe
   ```

6. Before we can scale-down the Kafka cluster again, we need to move the topics back tot he brokers 0, 1 and 2.
   Trigger the rebalance to _remove_ the brokers from the cluster by scheduling the topics on them to the remaining nodes of the cluster.
   Use the [`remove-brokers.file` file](./remove-brokers.yaml) file to do it.
   Notice the `.spec.mode` and `.spec.brokers` sections of the KafkaRebalance resource.
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.29.0/remove-brokers.yaml
   ```
   Wait for the proposal to become ready:
   ```
   kubectl get kafkarebalance remove-brokers -o wide -w
   ```
   And then approve it:
   ```
   kubectl annotate kafkarebalance remove-brokers strimzi.io/rebalance=approve
   ```
   And wait for the rebalance to complete:
   ```
   kubectl get kafkarebalance remove-brokers -o wide -w
   ```
   Once it is complete, check the topics again.
   You should see that some partitions now have replicas on the brokers 3 and 4.
   But no partitions should move between the nodes 0, 1 and 2.
   ```
   kubectl run kafka-clients -ti --image=quay.io/strimzi/kafka:0.29.0-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe
   ```

7. Now you can scale-down the Kafka cluster.
   Edit the Kafka CR using `kubectl edit kafka my-cluster` and set `.spec.kafka.replicas` to `3`.
   Wait until the brokers are moved.
   Check the topics again and you should see that nothing changed, they should be all in sync since they are all scheduled to the remaining brokers.
   ```
   kubectl run kafka-clients -ti --image=quay.io/strimzi/kafka:0.29.0-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe
   ```

## StrimziPodSets

_This is the same demo as last time. But with a lot of fixes and improvements inside!_ 😉

8. Check the existing Kafka cluster.
   with the `kubectl get statefulsets` command you should see the StatefulSets which it uses to manage the pods.

9. To use the `StrimziPodSets`, you have to enable the `UseStrimziPodSets` feature gate
   You can do it by setting the `STRIMZI_FEATURE_GATES` environment variable in the Cluster Operator to contain `+UseStrimziPodSets`.
   You can do that by editing the deployment, editing the Operator Hub subscription or using `kubectl set env` command.
   ```
   kubectl set env deployment/strimzi-cluster-operator STRIMZI_FEATURE_GATES=+UseStrimziPodSets
   ```

10. Watch the operator to roll all the pods.
    Once the rolling-update is finished, check the StatefulSets again with `kubectl get statefulsets`.
    They should not exist anymore.
    Now check the StrimziPodSets instead with `kubectl get strimzipodsets` and you should see the pod sets being used for ZooKeeper nodes and Kafka brokers

11. Try to delete the individual Kafka or ZooKeeper pods and check that they are recreated by the Strimzi operator.
    You can also try other things such as rolling-updates, scaling etc.

## (Experimental) KRaft mode

_This is experimental mode which should be used only for development and testing. Do not use this in production or try it out in your production cluster!_

12. Delete all existing Strimzi resource to start fresh
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```

13. Enable the `StrimziPodSets` and `UseKRaft` feature gates.
    You can do it by setting the `STRIMZI_FEATURE_GATES` environment variable in the Cluster Operator to contain `+UseStrimziPodSets,+UseKRaft`.
    You can do that by editing the deployment, editing the Operator Hub subscription or using `kubectl set env` command.
    ```
    kubectl set env deployment/strimzi-cluster-operator STRIMZI_FEATURE_GATES=+UseStrimziPodSets,+UseKRaft
    ``` 

14. Deploy the Kafka cluster using the [`./kraft.yaml`](./kraft.yaml) file.
    It follows the usual Strimzi example for deploying a Kafka cluster.
    But notice the commented out parts which are not supported.
    Notice also the `.spec.zookeeper` section which has to be present, although it is not really used.
    ```
    kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.29.0/kraft.yaml
    ```
    For a full list of features unsupported in the KRaft mode, visit the [docs](https://strimzi.io/docs/operators/latest/full/configuring.html#ref-operator-use-kraft-feature-gate-str).

15. Wait for the deployment to complete and check that ZooKeeper is indeed missing.
    Test the cluster by sending and receiving some messages.
    ```
    kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:0.29.0-rc1-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic
    kubectl run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.29.0-rc1-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
    ```

### Cleanup

16. Once done you can delete all the Strimzi resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
