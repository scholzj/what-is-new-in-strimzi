# What's new in Strimzi 0.32.0

## Prerequisites

1. Install Strimzi 0.32.0 using one of the available methods.

## Preparation

2. Deploy a new Apache Kafka cluster with Cruise Control, Authentication and Authorization enabled.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.32.0/kafka.yaml
   ```

## `cluster-ip` listener

3. The cluster we just deployed already uses the `cluster-ip` listener.
   You can check its configuration:
   ```yaml
    listeners:
      - name: tls
        port: 9093
        type: cluster-ip
        tls: true
        authentication:
          type: tls
   ```

4. List the services used by the Kafka cluster:
   ```
   kubectl get service
   ```
   And you should see (among other service) these 4 services used by the `cluster-ip` listener:
   ```
    NAME                             TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
    my-cluster-kafka-tls-0           ClusterIP      10.96.134.196    <none>          9093/TCP                     7m38s
    my-cluster-kafka-tls-1           ClusterIP      10.96.45.133     <none>          9093/TCP                     7m38s
    my-cluster-kafka-tls-2           ClusterIP      10.99.193.148    <none>          9093/TCP                     7m38s
    my-cluster-kafka-tls-bootstrap   ClusterIP      10.98.254.147    <none>          9093/TCP                     7m38s
    ...
   ```
   There are 3 per-broker services - `my-cluster-kafka-tls-0`, `my-cluster-kafka-tls-1`, and `my-cluster-kafka-tls-2` - one foreach broker.
   And one bootstrap service named `my-cluster-kafka-tls-bootstrap`.

5. Check the status of the Kafka resource:
   ```
   kubectl get kafka my-cluster -o yaml
   ```
   And you should see the bootstrap service also referenced there:
   ```yaml
    status:
      clusterId: q3xyOi2dSWGJA2TY9K7r7A
      conditions:
      - lastTransitionTime: "2022-11-01T21:25:45.622662Z"
        status: "True"
        type: Ready
      listeners:
      - addresses:
        - host: my-cluster-kafka-tls-bootstrap.myproject.svc
          port: 9093
        bootstrapServers: my-cluster-kafka-tls-bootstrap.myproject.svc:9093
        certificates:
        - |
          -----BEGIN CERTIFICATE-----
          ...
          -----END CERTIFICATE-----
        name: tls
        type: tls
      observedGeneration: 1
   ```

6. You can also check how the advertised hostnames are configured in the Kafka brokers:
   ```
   kubectl exec -ti my-cluster-kafka-0 -- cat /tmp/strimzi.properties | grep advertised.listeners
   ```
   And you should see something like this:
   ```
   advertised.listeners=CONTROLPLANE-9090://my-cluster-kafka-0.my-cluster-kafka-brokers.myproject.svc:9090,REPLICATION-9091://my-cluster-kafka-0.my-cluster-kafka-brokers.myproject.svc:9091,TLS-9093://my-cluster-kafka-tls-0.myproject.svc:9093
   ```
   Where the `cluster-ip` type listener is configured to advertise the address `my-cluster-kafka-tls-0.myproject.svc:9093`.

## Multiple operations per `ACLRule`

7. Deploy example clients using the Kafka cluster.
   You can use the [`clients.yaml`](./clients.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.32.0/clients.yaml
   ```
   Once the consumer and producer are up and running, you can check that they work by checking the logs:
   ```
   kubectl logs deployment/example-consumer -f
   ```
   You should see `Hello World` messages being received by the consumer

8. Check the `KafkaUser` definition in the [`clients.yaml`](./clients.yaml) file and notice how the ACL rights are specified:
   You can see that a single ACL rule now covers multiple ACL operations:
   ```yaml
    authorization:
      type: simple
      acls:
        - resource:
            type: topic
            name: my-topic
          operations: 
            - Create
            - Describe
            - Read
            - Write
        # ...
    ```

## Rebalance auto-approval

9. With a running cluster and some clients using it, we can try to trigger a rebalance.
   In one terminal window, start watching for the rebalance resources:
   ```
   kubectl get kafkarebalance -o wide -w
   ```

10. Now create a rebalance resource.
    You can use the [`rebalance.yaml`](./rebalance.yaml) file from this repository:
    ```
    kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.32.0/rebalance.yaml
    ```
    Notice the `strimzi.io/rebalance-auto-approval: true` annotation.

11. Get back to the terminal where you are watching the rebalances.
    You should see how the rebalance was automatically approved and and moved through the `ProposalReady` stage to `Rebalancing` and finally to `Ready` automatically:
    ```
    NAME           CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY
    my-rebalance   my-cluster
    my-rebalance   my-cluster   True
    my-rebalance   my-cluster                     True
    my-rebalance   my-cluster                                     True
    my-rebalance   my-cluster                                                   True
    ```

### Cleanup

12. Once done you can delete all the Strimzi resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
