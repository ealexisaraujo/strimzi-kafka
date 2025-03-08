To set up your Kafka cluster correctly from the beginning and avoid the issues we encountered, here's a complete guide:

### Step 1: Create the Kafka Cluster

Your YAML configuration is good for a development environment. For a production environment, you'd want at least 3 replicas and persistent storage, but this is fine for testing:

```bash
# Create namespace
kubectl create namespace kafka-strimzi

# Install Strimzi operator
kubectl apply -f https://strimzi.io/install/latest?namespace=kafka-strimzi -n kafka-strimzi

# Wait for the operator to be ready
kubectl wait --for=condition=ready pod -l name=strimzi-cluster-operator -n kafka-strimzi --timeout=300s

# Create Kafka cluster
kubectl apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka-strimzi
spec:
  kafka:
    version: 3.9.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    storage:
      type: ephemeral
    resources:
      requests:
        memory: 512Mi
        cpu: 250m
      limits:
        memory: 1Gi
        cpu: 500m
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  zookeeper:
    replicas: 1
    storage:
      type: ephemeral
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi
        cpu: 250m
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF

# Wait for Kafka to be ready
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka-strimzi
```

The key difference is that I've added the `config` section with appropriate replication factors for a single-broker cluster.

### Step 2: Create a Topic

```bash
kubectl apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka-strimzi
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 1
  replicas: 1
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824
EOF

# Wait for the topic to be created
kubectl wait kafkatopic/my-topic --for=condition=Ready --timeout=60s -n kafka-strimzi
```

### Step 3: Produce Messages

```bash
# Create a producer job
kubectl -n kafka-strimzi run kafka-producer --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --restart=Never -it -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic
```

Type your messages, one per line. Press Ctrl+D when done.

For automated message production:

```bash
# Send a single message
kubectl -n kafka-strimzi run kafka-producer-auto --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --restart=Never --rm -it -- bash -c 'echo "Automated message $(date)" | bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic'
```

### Step 4: Consume Messages

```bash
# Create a consumer job
kubectl -n kafka-strimzi run kafka-consumer --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning

# View the consumer logs
kubectl -n kafka-strimzi logs -f kafka-consumer
```

For a one-time consumption:

```bash
kubectl -n kafka-strimzi run kafka-consumer-once --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --restart=Never --rm -it -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning --max-messages 10
```

### Step 5: Create a Java Application (Optional)

If you want to create a Java application to produce/consume messages:

```java
// Producer example
Properties props = new Properties();
props.put("bootstrap.servers", "my-cluster-kafka-bootstrap.kafka-strimzi.svc:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

Producer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("my-topic", "key", "value"));
producer.close();

// Consumer example
Properties props = new Properties();
props.put("bootstrap.servers", "my-cluster-kafka-bootstrap.kafka-strimzi.svc:9092");
props.put("group.id", "my-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

Consumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("my-topic"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("offset = %d, key = %s, value = %s%n",
                          record.offset(), record.key(), record.value());
    }
}
```

### Step 6: Clean Up

To clean up all resources:

```bash
# Delete Kafka topics
kubectl delete kafkatopic --all -n kafka-strimzi

# Delete Kafka users (if any)
kubectl delete kafkauser --all -n kafka-strimzi

# Delete Kafka cluster
kubectl delete kafka --all -n kafka-strimzi

# Wait for Kafka resources to be deleted
kubectl wait --for=delete pod -l strimzi.io/cluster=my-cluster -n kafka-strimzi --timeout=300s

# Delete the Strimzi operator
kubectl delete deployment strimzi-cluster-operator -n kafka-strimzi

# Delete Strimzi RBAC resources
kubectl delete clusterroles strimzi-cluster-operator-namespaced strimzi-cluster-operator-global strimzi-kafka-broker strimzi-entity-operator -n kafka-strimzi --ignore-not-found
kubectl delete clusterrolebindings strimzi-cluster-operator strimzi-cluster-operator-kafka-broker-delegation strimzi-cluster-operator-entity-operator-delegation -n kafka-strimzi --ignore-not-found

# Delete Strimzi CRDs
kubectl delete crd kafkas.kafka.strimzi.io kafkatopics.kafka.strimzi.io kafkausers.kafka.strimzi.io kafkaconnects.kafka.strimzi.io kafkamirrormakers.kafka.strimzi.io kafkabridges.kafka.strimzi.io kafkaconnectors.kafka.strimzi.io kafkamirrormaker2s.kafka.strimzi.io kafkarebalances.kafka.strimzi.io kafkanodepools.kafka.strimzi.io strimzipodsets.core.strimzi.io --ignore-not-found

# Delete any remaining resources
kubectl delete pod,service,configmap,secret,pvc --all -n kafka-strimzi

# Delete the namespace
kubectl delete namespace kafka-strimzi
```



### Troubleshooting Tips

If you encounter issues:

1. Check if the Kafka cluster is ready:
   ```bash
   kubectl get kafka -n kafka-strimzi
   ```

2. Check if the topic exists:
   ```bash
   kubectl -n kafka-strimzi run kafka-topics-list --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --restart=Never --rm -it -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --list
   ```

3. Check consumer groups:
   ```bash
   kubectl -n kafka-strimzi run kafka-consumer-groups --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --restart=Never --rm -it -- bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --list
   ```

4. Create a debug pod for troubleshooting:
   ```bash
   kubectl -n kafka-strimzi run kafka-debug --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --restart=Never -- sleep 3600
   kubectl -n kafka-strimzi exec -it kafka-debug -- bash
   ```

The key to avoiding the issues we encountered is setting the correct replication factors in the Kafka configuration for a single-broker setup.

Limitations of Single-Broker Setup
No high availability: If the broker goes down, the entire Kafka cluster is unavailable
No data replication: Data is stored on a single broker, increasing the risk of data loss
Limited scalability: Cannot distribute load across multiple brokers
Not suitable for production: This setup should only be used for development and testing