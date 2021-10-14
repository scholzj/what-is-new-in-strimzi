# What's new in Strimzi 0.26.0

## Install Strimzi

* Install Strimzi 0.26.0 using one of the available methods

## Kafka Connect Build

### Preparation

1. Deploy a new Apache Kafka 3.0.0 cluster.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.26.0/kafka.yaml
   ```

### Deploy Kafka Connect with `maven` type artifact

2. Check the [`connect.yaml`](./connect.yaml) file with the Kafka Connect configuration.
   Notice the `camel-timer-connector` which uses the new `maven` artifact type and will download the connector directly from Maven repositories.
   ```yaml
   - name: camel-timer-connector
     artifacts:
       - type: maven
         group: org.apache.camel.kafkaconnector
         artifact: camel-timer-kafka-connector
         version: 0.11.0
    ```
3. Before deploying the Kafka Connect cluster, you need to configure the container repository where the new container with the additional plugins should be pushed.
   If you have some registry where you can push without authentication, you can just update the image address in the `output` section of [`connect.yaml`](./connect.yaml):
   ```yaml
    output:
      type: docker
      image: my-registry:5000/myproject/kafka-connect:latest
   ```
   If your registry requires authentication, you need to first create the secret with credentials.
   The secret should look similar to this:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: docker-credentials
   type: kubernetes.io/dockerconfigjson
   data:
     .dockerconfigjson: Cg==
   ```
   You can follow the [Kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry) for more details how to create it.
   Once you have the secret, you have to reference it in the `output` section of [`connect.yaml`](./connect.yaml):
   ```yaml
   output:
     type: docker
     image: docker.io/my-org/kafka-connect:latest
     pushSecret: docker-credentials
   ```
4. Deploy the Kafka Connect cluster:
   ```
   kubectl apply -f connect.yaml
   ```
   Wait until the build is finished and the Connect cluster is deployed:
   ```
   kubectl wait kafkaconnect/my-connect --for=condition=Ready --timeout=300s
   ```
5. Check the available connectors in the status of the `KafkaConnect` custom resource:
   ```
   kubectl get kafkaconnect my-connect -o yaml
   ```
   You should see something like this:
   ```yaml
   # ...
   status:
     conditions:
      - lastTransitionTime: "2021-10-14T15:30:22.766432Z"
        status: "True"
        type: Ready
     connectorPlugins:
       - class: cz.scholz.kafka.connect.echosink.EchoSinkConnector
         type: sink
         version: 1.2.0
       - class: org.apache.camel.kafkaconnector.CamelSinkConnector
         type: sink
         version: 0.11.0
       - class: org.apache.camel.kafkaconnector.CamelSourceConnector
         type: source
         version: 0.11.0
       - class: org.apache.camel.kafkaconnector.timer.CamelTimerSourceConnector
         type: source
         version: 0.11.0
       - class: org.apache.kafka.connect.file.FileStreamSinkConnector
         type: sink
         version: 3.0.0
       - class: org.apache.kafka.connect.file.FileStreamSourceConnector
         type: source
         version: 3.0.0
       - class: org.apache.kafka.connect.mirror.MirrorCheckpointConnector
         type: source
         version: "1"
       - class: org.apache.kafka.connect.mirror.MirrorHeartbeatConnector
         type: source
         version: "1"
       - class: org.apache.kafka.connect.mirror.MirrorSourceConnector
         type: source
         version: "1"
     labelSelector: strimzi.io/cluster=my-connect,strimzi.io/name=my-connect-connect,strimzi.io/kind=KafkaConnect
     observedGeneration: 1
     replicas: 1
     url: http://my-connect-connect-api.myproject.svc:8083
   ```
   Notice the `CamelTimerSourceConnector` connector.
   We can also check inside the new container that the connector was pulled including all dependencies:
   ```
   kubectl exec -ti deployment/my-connect-connect -- ls plugins/camel-timer-connector/1a9cd13e
   ```
   Which should show output similar or equal to this:
   ```
   annotations-13.0.jar
   apicurio-registry-common-1.3.2.Final.jar
   apicurio-registry-rest-client-1.3.2.Final.jar
   apicurio-registry-utils-converter-1.3.2.Final.jar
   apicurio-registry-utils-serde-1.3.2.Final.jar
   avro-1.10.2.jar
   camel-api-3.11.1.jar
   camel-base-3.11.1.jar
   camel-base-engine-3.11.1.jar
   camel-core-engine-3.11.1.jar
   camel-core-languages-3.11.1.jar
   camel-core-model-3.11.1.jar
   camel-core-processor-3.11.1.jar
   camel-core-reifier-3.11.1.jar
   camel-direct-3.11.1.jar
   camel-jackson-3.11.1.jar
   camel-kafka-3.11.1.jar
   camel-kafka-connector-0.11.0.jar
   camel-main-3.11.1.jar
   camel-management-api-3.11.1.jar
   camel-seda-3.11.1.jar
   camel-support-3.11.1.jar
   camel-timer-3.11.1.jar
   camel-timer-kafka-connector-0.11.0.jar
   camel-util-3.11.1.jar
   commons-compress-1.20.jar
   connect-api-2.8.0.jar
   connect-json-2.6.0.jar
   connect-transforms-2.8.0.jar
   converter-jackson-2.9.0.jar
   jackson-annotations-2.12.3.jar
   jackson-core-2.12.3.jar
   jackson-databind-2.12.3.jar
   jackson-dataformat-avro-2.12.2.jar
   jackson-datatype-jdk8-2.12.2.jar
   javax.annotation-api-1.3.2.jar
   javax.ws.rs-api-2.1.1.jar
   jboss-jaxrs-api_2.1_spec-2.0.1.Final.jar
   jctools-core-3.3.0.jar
   kafka-clients-2.8.0.jar
   kotlin-reflect-1.3.20.jar
   kotlin-stdlib-1.3.20.jar
   kotlin-stdlib-common-1.3.20.jar
   lz4-java-1.7.1.jar
   medeia-validator-core-1.1.1.jar
   medeia-validator-jackson-1.1.1.jar
   okhttp-4.8.1.jar
   okio-2.7.0.jar
   protobuf-java-3.13.0.jar
   retrofit-2.9.0.jar
   slf4j-api-1.7.30.jar
   snappy-java-1.1.8.1.jar
   zstd-jni-1.4.9-1.jar  
   ```
6. With the connector plugins ready, we can now create the connector instances using the file [`connectors.yaml`](./connectors.yaml) which creates:
     * Kafka topic named `timer-topic`
     * Connector named `camel-timer-connector` which will send a new message every second
     * Connector names `echo-sink-timer-connector` which will read the messages from the first connector and print their content into the log
   You can deploy it using:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.26.0/connectors.yaml
   ```
7. Once the connectors are running, you can check the Connect logs:
   ```
   kubectl logs -f deployment/my-connect-connect
   ```
   And you should see the messages sent by the timer connector:
   ```
   2021-10-14 15:48:21,569 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226501519}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:22,570 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226502520}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:23,571 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226503520}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:24,571 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226504521}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:25,571 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226505521}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:26,572 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226506522}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   ```

### Cleanup

8. Once done you can delete all the Strimzi resources used during the demo:
   ```
   kubectl delete $(kubectl get strimzi -o name)
   ```
   And uninstall the operator.
   If you created the secret with the container registry credentials, don't forget to delete it as well.
