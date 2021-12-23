# What's new in Strimzi 0.27.0

## Install Strimzi

* Install Strimzi 0.27.0 using one of the available methods

## Control Plane listener

### Preparation

1. Deploy a new Apache Kafka 3.0.0 cluster.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.27.0/kafka.yaml
   ```

### Check that the Control Plane Listener is used

2. Check that the Control Plane Listener on port 9090 is being used in the Kafka brokers:
   ```
   kubectl logs my-cluster-kafka-0 | grep acceptor
   ```
   You should see something like this:
   ```
   2021-12-23 16:21:48,137 INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Created control-plane acceptor and processor for endpoint : ListenerName(CONTROLPLANE-9090) (kafka.network.SocketServer) [main]
   ...
   2021-12-23 16:21:49,288 INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Started control-plane acceptor and processor(s) for endpoint : ListenerName(CONTROLPLANE-9090) (kafka.network.SocketServer) [main]
   ...
   ```
3. You can also check the configuration file of the Kafka broker:
   ```
   kubectl exec -ti my-cluster-kafka-0 -- cat /tmp/strimzi.properties | grep control.plane
   ```

### Disable the Control Plane Listener feature gate

4. Disable the `ControlPlaneListener` feature gate.
   You can do it by setting the `STRIMZI_FEATURE_GATES` environment variable in the Cluster Operator to contain `-ControlPlaneListener`.
   You can do that by editing the deployment, editing the Operator Hub subscription or using `kubectl set env` command.
   ```
   kubectl set env deployment/strimzi-cluster-operator STRIMZI_FEATURE_GATES=-ControlPlaneListener
   ```
5. Wait for the Kafka brokers to roll.
6. Check that the Control Plane Listener on port 9090 is not used in anymore:
   ```
   kubectl logs my-cluster-kafka-0 | grep control-plane
   ```
   You can also check the configuration file of the Kafka broker:
   ```
   kubectl exec -ti my-cluster-kafka-0 -- cat /tmp/strimzi.properties | grep control.plane
   ```

### Cleanup

7. Once done you can delete all the Strimzi resources used during the demo:
   ```
   kubectl delete $(kubectl get strimzi -o name)
   ```
   And uninstall the operator.
