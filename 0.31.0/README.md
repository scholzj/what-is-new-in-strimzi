# What's new in Strimzi 0.31.0

## Prerequisites

1. Install Strimzi 0.31.0 using one of the available methods

## Leader Election

2. Check the configuration of the deployed Cluster Operator and notice the options related to the leader election:
   ```yaml
            - name: STRIMZI_LEADER_ELECTION_ENABLED
              value: "true"
            - name: STRIMZI_LEADER_ELECTION_LEASE_NAME
              value: "strimzi-cluster-operator"
            - name: STRIMZI_LEADER_ELECTION_LEASE_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: STRIMZI_LEADER_ELECTION_IDENTITY
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
   ```
   The leader election is enabled by default, even through only one replica is used by default.
   Check the docs for all available options.

3. Scale the Cluster Operator to use 2 replicas.
   The exact command might depend on how you installed Strimzi.
   You can use for example the `kubectl scale deployment` command
   ```
   kubectl scale deployment strimzi-cluster-operator --replicas 2
   ```

4. Check the newly created operator pod and its log.
   You should see that it is in the waiting mode:
   ```
   ...
   2022-11-01 20:25:35 INFO  Main:245 - Waiting to become a leader
   2022-11-01 20:25:36 INFO  LeaderElectionManager:119 - The new leader is strimzi-cluster-operator-6c8df5b94b-d4pjz
   ```

5. Try to kill the original pod which is currently a leader.
   You can do it for example using the `kubectl delete pod ...` command.
   That should trigger the leader election.
   In the log of the Pod which is the new leader, you should see following messages after which the pod starts managing the operands:
   ```
   2022-11-01 20:27:23 INFO  LeaderElectionManager:100 - Started being a leader
   2022-11-01 20:27:23 INFO  Main:233 - I'm the new leader
   ...
   ```
   _Note: There is no exact guarantee which of the remaining pods becomes the new leader._
   _It does not have to be the oldest one._
   _It can be also the pod started as a replacement for the pod we just deleted._

## Pod Security Providers

6. Enable the `restricted` Pod Security Provider.
   You can do it by setting the `STRIMZI_POD_SECURITY_PROVIDER_CLASS` environment variable in the Cluster Operator to contain `restricted`.
   * If you deployed it using YAML files, you can just edit the deployment file and add:
     ```yaml
     - name: STRIMZI_POD_SECURITY_PROVIDER_CLASS
       value: "restricted"
     ```
     Or use the `kubectl set env` command.
     ```
     kubectl set env deployment/strimzi-cluster-operator STRIMZI_POD_SECURITY_PROVIDER_CLASS=restricted
     ```
   * If you use Operator Hub, you can set it in the `Subscription` resource:
     ```yaml
     spec:
       # ...
       config:
         env:
           - name: STRIMZI_POD_SECURITY_PROVIDER_CLASS
             value: "restricted"
     ```

7. Deploy a new Apache Kafka cluster.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.31.0/kafka.yaml
   ```

6. Wait for the Kafka cluster to get ready and then check the security context of the different pods.
   You should see the following security context for all containers:
   ```yaml
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
   ```
   You can use `kubectl` to look at it.
   Either using:
   ```
   kubectl get pod my-cluster-kafka-0 -o yaml
   ```
   Or for example using:
   ```
   kubectl get pod my-cluster-kafka-0 -o jsonpath='{.spec.containers[0].securityContext}' | jq
   ```

### Cleanup

9. Once done you can delete all the Strimzi resources used during the demo:
   ```
   kubectl delete $(kubectl get strimzi -o name)
   ```
   And uninstall the operator.
