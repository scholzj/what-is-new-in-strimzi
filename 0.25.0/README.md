# What's new in Strimzi 0.25.0

## Install Strimzi

* Install Strimzi 0.25.0 using one of the available methods

## User Operator

### Preparation

1. Deploy a Kafka cluster with different listeners using TLS and SCRAM-SHA-512 authentication and User Operator enabled.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f kafka.yaml
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
     kubectl apply -f scram-sha-user.yaml
     ```
4. Check the created user:
   ```
   kubectl get kafkauser scram-sha-user -o yaml
   ```
   * The condition should be ready:
     ```
     TODO
     ```
   * The custom password should also in the user secret:
     ```
     kubectl get secret scram-sha-user -o jsonpath='{.data.password}' | base64 -d
     ```
   * We can also check that we can connect with a client:
     ```
     TODO
     ```
5. You can also try what happens when the secret does not exist.
   The User Operator will not generate random password but wait for the secret to be created.
   You can try that by creating [`scram-sha-user2.yaml`](./scram-sha-user.yaml):
   ```
   kubectl apply -f scram-sha-user2.yaml
   ```
6. Create the secret and check that the next reconciliation creates the user:
   ```
   kubectl create secret generic scram-sha-user2-custom-password --from-literal=customPassword='myOtherRandomAndSecretPassword'
   ```

### `tls-external` authentication

7. Create a `KafkaUser` with `tls-external` authentication type.
   You can use the [`tls-external-user.yaml`](./tls-external-user.yaml) file from this repository:
   ```
   kubectl apply -f tls-external-user.yaml
   ```
8. Check that the user was created, but without generating the certificate
   * Check that the user secret was not created:
     ```
     kubectl get secret tls-external-user
     ```
   * Check that the ACLs were set for the user `CN=tls-external-user`:
     ```
     TODO
     ```

### Cleanup

9. Delete the users and the Kafka cluster after the demo:
   ```
   kubectl delete kafka my-cluster
   kubectl delete kafkauser tls-external-user scram-sha-user scram-sha-user2
   kubectl delete secret scram-sha-user-custom-password scram-sha-user2-custom-password
   ```

## Disabling Network Policies