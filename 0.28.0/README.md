# What's new in Strimzi 0.28.0

## Prerequisites

* Kubernetes cluster with FIPS mode enabled and with support for Load Balancers

## Running on FIPS enabled Kubernetes clusters

1. Install Strimzi 0.28.0 using one of the available methods

2. Deploy a new Apache Kafka 3.1.0 cluster.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.28.0/kafka.yaml
   ```

3. You should see that the Strimzi Cluster Operator is crash-looping with an error similar to this:
   ```
   Exception in thread "main" io.fabric8.kubernetes.client.KubernetesClientException: An error has occurred.
      at io.fabric8.kubernetes.client.KubernetesClientException.launderThrowable(KubernetesClientException.java:103)
      at io.fabric8.kubernetes.client.KubernetesClientException.launderThrowable(KubernetesClientException.java:97)
      at io.fabric8.kubernetes.client.utils.HttpClientUtils.applyCommonConfiguration(HttpClientUtils.java:214)
      at io.fabric8.kubernetes.client.okhttp.OkHttpClientFactory.createHttpClient(OkHttpClientFactory.java:89)
      at io.fabric8.kubernetes.client.utils.HttpClientUtils.createHttpClient(HttpClientUtils.java:164)
      at io.fabric8.kubernetes.client.BaseClient.<init>(BaseClient.java:48)
      at io.fabric8.kubernetes.client.BaseClient.<init>(BaseClient.java:40)
      at io.fabric8.kubernetes.client.BaseKubernetesClient.<init>(BaseKubernetesClient.java:151)
      at io.fabric8.kubernetes.client.DefaultKubernetesClient.<init>(DefaultKubernetesClient.java:34)
      at io.strimzi.operator.cluster.Main.main(Main.java:75)
   Caused by: java.security.KeyStoreException: sun.security.pkcs11.wrapper.PKCS11Exception: CKR_SESSION_READ_ONLY
      at jdk.crypto.cryptoki/sun.security.pkcs11.P11KeyStore.engineSetEntry(P11KeyStore.java:1049)
      at jdk.crypto.cryptoki/sun.security.pkcs11.P11KeyStore.engineSetCertificateEntry(P11KeyStore.java:515)
      at java.base/java.security.KeyStore.setCertificateEntry(KeyStore.java:1235)
      at io.fabric8.kubernetes.client.internal.CertUtils.createTrustStore(CertUtils.java:100)
      at io.fabric8.kubernetes.client.internal.CertUtils.createTrustStore(CertUtils.java:74)
      at io.fabric8.kubernetes.client.internal.SSLUtils.trustManagers(SSLUtils.java:140)
      at io.fabric8.kubernetes.client.internal.SSLUtils.trustManagers(SSLUtils.java:90)
      at io.fabric8.kubernetes.client.utils.HttpClientUtils.applyCommonConfiguration(HttpClientUtils.java:203)
      ... 7 more
   Caused by: sun.security.pkcs11.wrapper.PKCS11Exception: CKR_SESSION_READ_ONLY
      at jdk.crypto.cryptoki/sun.security.pkcs11.wrapper.PKCS11.C_CreateObject(Native Method)
      at jdk.crypto.cryptoki/sun.security.pkcs11.wrapper.PKCS11$FIPSPKCS11.C_CreateObject(PKCS11.java:1950)
      at jdk.crypto.cryptoki/sun.security.pkcs11.P11KeyStore.storeCert(P11KeyStore.java:1567)
      at jdk.crypto.cryptoki/sun.security.pkcs11.P11KeyStore.engineSetEntry(P11KeyStore.java:1045)
      ... 14 more
   ```
   And no Kafka cluster is deployed.

4. Disable the FIPS mode by setting the `FIPS_MODE` environment variable in the Cluster Operator to `disabled`.
   You can do that by editing the deployment, editing the Operator Hub subscription or using `kubectl set env` command.
   ```
   kubectl set env deployment/strimzi-cluster-operator FIPS_MODE=disabled
   ```
5. Watch the operator pod to recreate and start deploying the Kafka cluster.

## StrimziPodSets

6. Check the existing Kafka cluster.
   with the `kubectl get statefulsets` command you should see the StatefulSets which it uses to manage the pods.

7. To use the `StrimziPodSets`, you have to enable the `UseStrimziPodSets` feature gate
   You can do it by setting the `STRIMZI_FEATURE_GATES` environment variable in the Cluster Operator to contain `+UseStrimziPodSets`.
   You can do that by editing the deployment, editing the Operator Hub subscription or using `kubectl set env` command.
   ```
   kubectl set env deployment/strimzi-cluster-operator STRIMZI_FEATURE_GATES=+UseStrimziPodSets
   ```

8. Watch the operator to roll all the pods.
   Once the rolling-update is finished, check the StatefulSets again with `kubectl get statefulsets`.
   They should not exist anymore.
   Now check the StrimziPodSets instead with `kubectl get strimzipodsets` and you should see the pod sets being used for ZooKeeper nodes and Kafka brokers

9. Try to delete the individual Kafka or ZooKeeper pods and check that they are recreated by the Strimzi operator.
   You can also try other things such as rolling-updates, scaling etc.

## Disabling bootstrap load balancers

10. Edit your Kafka cluster and add a `loadbalancer` type listener.
    Set the `createBootstrapService` option to `false` to avoid creating the bootstrap load balancer.
    ```yaml
      - name: lb
        port: 9094
        type: loadbalancer
        tls: true
        configuration:
          createBootstrapService: false
    ```

11. Wait for the cluster to roll and check the services / load balancers.
    You should see only the per-broker load balancers.

### Cleanup

12. Once done you can delete all the Strimzi resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
