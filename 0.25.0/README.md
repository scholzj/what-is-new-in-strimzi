# What's new in Strimzi 0.25.0

## Install Strimzi

* Install Strimzi 0.25.0 using one of the available methods

## User Operator

### Preparation

1. Deploy a Kafka cluster with different listeners using TLS and SCRAM-SHA-512 authentication and User Operator enabled.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.25.0/kafka.yaml
   ```

### Custom Password

2. Create the secret with the desired password.
   The secret should have a name different from the regular `KafkaUser` secret:
   ```
   kubectl create secret generic scram-sha-user-custom-password --from-literal=customPassword='myRandomAndSecretPassword'
   ```
3. Create the `KafkaUser` and reference the password from the secret created in the previous step:
   ```yaml
   # ...
   authentication:
     type: scram-sha-512
     password:
       valueFrom:
         secretKeyRef:
           name: scram-sha-user-custom-password
           key: customPassword
   # ...
   ```
   * You can use the [`scram-sha-user.yaml`](./scram-sha-user.yaml) file from this repository:
     ```
     kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.25.0/scram-sha-user.yaml
     ```
4. Check the created user:
   ```
   kubectl get kafkauser scram-sha-user -o yaml
   ```
   * The condition should be ready:
     ```yaml
     status:
       conditions:
         - lastTransitionTime: "2021-08-09T19:20:29.029522Z"
           status: "True"
           type: Ready
     ```
   * The custom password should also in the user secret:
     ```
     kubectl get secret scram-sha-user -o jsonpath='{.data.password}' | base64 -d
     ```
   * We can also check that we can connect with a client:
     * List metadata
       ```
       kubectl run kafka-metadata -ti --image=edenhill/kafkacat:1.6.0 --rm=true --restart=Never -- -b my-cluster-kafka-bootstrap:9092 -X security.protocol=SASL_PLAINTEXT -X sasl.mechanism=SCRAM-SHA-512 -X sasl.username=scram-sha-user -X sasl.password=myRandomAndSecretPassword -L
       ```
     * Produce and consume messages
       ```
       kubectl run kafka-producer -ti --image=edenhill/kafkacat:1.6.0 --rm=true --restart=Never -- -b my-cluster-kafka-bootstrap:9092 -X security.protocol=SASL_PLAINTEXT -X sasl.mechanism=SCRAM-SHA-512 -X sasl.username=scram-sha-user -X sasl.password=myRandomAndSecretPassword -t my-topic -P
       kubectl run kafka-consumer -ti --image=edenhill/kafkacat:1.6.0 --rm=true --restart=Never -- -b my-cluster-kafka-bootstrap:9092 -X security.protocol=SASL_PLAINTEXT -X sasl.mechanism=SCRAM-SHA-512 -X sasl.username=scram-sha-user -X sasl.password=myRandomAndSecretPassword -C -t my-topic -o beginning
       ```
5. You can also try what happens when the secret does not exist.
   The User Operator will not generate random password but wait for the secret to be created.
   You can try that by creating [`scram-sha-user2.yaml`](./scram-sha-user.yaml):
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.25.0/scram-sha-user2.yaml
   ```
6. The user should fail to create with the following condition:
   ```yaml
   status:
     conditions:
       - lastTransitionTime: "2021-08-09T19:16:51.116530Z"
         message: Secret scram-sha-user-custom-password with requested user password does not exist.
         reason: InvalidResourceException
         status: "True"
         type: NotReady
   ```
7. Create the secret and check that the next reconciliation creates the user:
   ```
   kubectl create secret generic scram-sha-user2-custom-password --from-literal=customPassword='myOtherRandomAndSecretPassword'
   ```

### `tls-external` authentication

8. Create a `KafkaUser` with `tls-external` authentication type.
   You can use the [`tls-external-user.yaml`](./tls-external-user.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.25.0/tls-external-user.yaml
   ```
9. Check that the user was created, but without generating the certificate
   * Check that the user secret was not created:
     ```
     kubectl get secret tls-external-user
     ```
   * Check that the ACLs were set for the user `CN=tls-external-user`:
     ```
     kubectl exec -ti my-cluster-zookeeper-0 -- bin/zookeeper-shell.sh localhost:12181 get /kafka-acl/Topic/my-topic
     ```

### Cleanup

10. Delete the users and the Kafka cluster after the demo:
   ```
   kubectl delete kafka my-cluster
   kubectl delete kafkauser tls-external-user scram-sha-user scram-sha-user2
   kubectl delete secret scram-sha-user-custom-password scram-sha-user2-custom-password
   ```

## Disabling Network Policies

1. Edit the Cluster Operator deployment and set the environment variable `STRIMZI_NETWORK_POLICY_GENERATION` to `false`.
   * If you deployed it using YAML files, you can just edit the deployment file and add:
     ```yaml
     - name: STRIMZI_NETWORK_POLICY_GENERATION
       value: "false"
     ```
   * If you use Operator Hub, you can set it in the `Subscription` resource:
     ```yaml
     spec:
     # ...
     config:
       env:
         - name: STRIMZI_NETWORK_POLICY_GENERATION
         value: "false"
     ```
2. Deploy a Kafka cluster.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.25.0/kafka.yaml
   ```
3. Check the effect of the disabled Network Policies
   * No network policies are created by the operator:
     ```
     kubectl get networkpolicies
     ```
   * If you create your own Network Policy, the operator will not overwrite it.
     You can use [`network-policy.yaml`](./network-policy.yaml) file from this repository as an example:
     ```
     kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.25.0/network-policy.yaml
     kubectl get networkpolicy -w
     ```
