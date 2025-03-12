## Config Producer  
### Using confluent-kafka (recommended)
```python
from confluent_kafka import Producer

config = {
    'bootstrap.servers': 'kafka1:9092,kafka2:9092',  # List broker
    'acks': 'all',                                   # Ensure successful writes to all replicas
    'compression.type': 'snappy',                   # Data compression (snappy, lz4, gzip)
    'linger.ms': 20,                                # Wait up to 20ms for batching
    'batch.size': 1048576,                          # 1MB/batch
    'max.in.flight.requests.per.connection': 5,     # Maximum number of unACKed requests
    'retries': 10,                                  # Number of retries on error
    'delivery.timeout.ms': 120000,                  # Timeout for sending message (2 minutes)
    'enable.idempotence': True,                     # Guaranteed no duplicates
    'message.timeout.ms': 30000,                    # Timeout for each message
    'queue.buffering.max.messages': 100000,         # Maximum number of messages in buffer
}

producer = Producer(config)
```

### Using kafka-python
```python
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers='kafka1:9092,kafka2:9092',
    acks='all',
    compression_type='snappy',
    linger_ms=20,
    batch_size=1048576,
    max_in_flight_requests_per_connection=5,
    retries=10,
    request_timeout_ms=120000,
    api_version=(2, 8, 1)  # Use the appropriate Kafka version
)
```
## Best Practices when implementing Producer
1. Asynchronous Processing with Callbacks
    - Use callback to confirm successful message sending or handle errors
    ```python
    def delivery_report(err, msg):
        if err:
            print(f"Send error: {err}")
            # Retry or log into Dead Letter Queue (DLQ)
        else:
            print(f"Send success: {msg.topic()}/{msg.partition()}/{msg.offset()}")

    # Send message with callback
    producer.produce(
        topic='high-load-topic',
        key=str(uuid.uuid4()).encode(),
        value=json.dumps(data).encode(),
        callback=delivery_report
    )

    # Ensure all messages are sent before termination
    producer.flush()
    ```
2. Batch and Compression
    - Increase throughput by optimizing batch.size and linger.ms.
    - Compress data (snappy or lz4) to reduce network I/O.
3. Idempotence and Transaction
    - Turn on `enable.idempotence=True` to avoid duplicate messages on retry.
    - Use transactions if you need to send atomic messages to multiple topics:
    ```python
    producer.init_transactions()
    producer.begin_transaction()
    try:
        producer.produce(...)
        producer.commit_transaction()
    except Exception as e:
        producer.abort_transaction()
    ```
### Optimizing Kafka system
1. Broker Configuration
    - Increase the number of partitions (eg: 10-100 partitions/topic) to distribute the load.
    - Replication factor ≥ 3 to ensure fault tolerance.
    - Increase num.io.threads and num.network.threads in server.properties.
2. Log Retention
    - Set `log.retention.bytes` and `log.retention.hours` accordingly
    ```conf
    log.retention.bytes=10737418240  # 10GB/topic
    log.retention.hours=168          # 7 days
    ```
