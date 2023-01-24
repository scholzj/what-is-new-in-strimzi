# What's new in Strimzi 0.33.0

## Prerequisites

1. Install Strimzi 0.33.0 using one of the available methods.

## Preparation

2. Deploy a new Apache Kafka cluster and wait until it is ready.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.33.0/kafka.yaml
   ```

## Connector auto-restarting

3. Create a new topic `my-topic`.
   You can use the [`topic.yaml`](./topic.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.33.0/topic.yaml
   ```

4. Deploy a new Connect cluster and wait until it is ready.
   Use the [`connect.yaml`](./connect.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.33.0/connect.yaml
   ```
   The Connect deployment adds the [Echo Sink connector](https://github.com/scholzj/echo-sink) which we will use to demonstrate the auto-restarting feature.
   It also enables the connector operator.

5. Create the connector and configure it to fail after receiving 5 messages.
   You can use the [`connector.yaml`](./connector.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-strimzi/main/0.33.0/connector.yaml
   ```
   It enables the auto-restart feature in `.spec.autoRestart`.
   Also notice the `fail.task.after.records: 5` settings in the connector configuration which tells it to fail after 5 records.
   The connector and its task should start and run without any issues, because it is not receiving any messages yet.

6. Send first five messages to the `my-topic` topic:
   ```
   kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:0.33.0-kafka-3.3.2 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic
   If you don't see a command prompt, try pressing enter.
   >Hello World 1
   >Hello World 2
   >Hello World 3
   >Hello World 4
   >Hello World 5
   ```
   After the fifth message, you should see in the Connect log that the connector task fails:
   ```
   2023-01-22 23:02:52,246 WARN [echo-sink|task-0] Failing as requested after 5 records (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-0]
   2023-01-22 23:02:52,246 ERROR [echo-sink|task-0] WorkerSinkTask{id=echo-sink-0} Task threw an uncaught and unrecoverable exception. Task is being killed and will not recover until manually restarted. Error: Intentional task failure after receiving 5 records. (org.apache.kafka.connect.runtime.WorkerSinkTask) [task-thread-echo-sink-0]
   java.lang.RuntimeException: Intentional task failure after receiving 5 records.
     at cz.scholz.kafka.connect.echosink.EchoSinkTask.put(EchoSinkTask.java:131)
     at org.apache.kafka.connect.runtime.WorkerSinkTask.deliverMessages(WorkerSinkTask.java:581)
     at org.apache.kafka.connect.runtime.WorkerSinkTask.poll(WorkerSinkTask.java:333)
     at org.apache.kafka.connect.runtime.WorkerSinkTask.iteration(WorkerSinkTask.java:234)
     at org.apache.kafka.connect.runtime.WorkerSinkTask.execute(WorkerSinkTask.java:203)
     at org.apache.kafka.connect.runtime.WorkerTask.doRun(WorkerTask.java:189)
     at org.apache.kafka.connect.runtime.WorkerTask.run(WorkerTask.java:244)
     at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539)
     at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
     at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
     at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
     at java.base/java.lang.Thread.run(Thread.java:833)
   ```

7. Wait until the next reconciliation and check that the operator restarted the task.
   You should see log messages similar to the following in the operator log:
   ```
   2023-01-22 23:04:24 INFO  AbstractConnectOperator:692 - Reconciliation #73(timer) KafkaConnect(myproject/my-connect): Auto restarting connector echo-sink
   2023-01-22 23:04:24 INFO  AbstractConnectOperator:696 - Reconciliation #73(timer) KafkaConnect(myproject/my-connect): Restarted connector echo-sink
   ```
   After the restart you should also see the connector status to be updated:
   ```yaml
   # ...
   status:
     # ...
     autoRestart:
       count: 1
       lastRestartTimestamp: "2023-01-22T23:04:24.386944356Z"
   ```

8. Try to repeat this multiple times by sending more messages to the `my-topic` topic.

9. After you stop sending messages, the connector should recover and after some time, the auto-restart status should reset back to 0.
   You should see log messages similar to the following in the operator log:
   ```
   2023-01-22 23:26:24 INFO  AbstractConnectOperator:660 - Reconciliation #100(timer) KafkaConnect(myproject/my-connect): Resetting the auto-restart status of connector echo-sink
   ```
   And the `.status.autoRestart` section will be removed from the `KafkaConnect` resource

### Cleanup

10. Once done you can delete all the Strimzi resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
