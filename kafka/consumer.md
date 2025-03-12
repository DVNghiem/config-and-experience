## Config Consumer 
### Using confluent-kafka (recommended)
```python
from confluent_kafka import Consumer, KafkaException

config = {
    'bootstrap.servers': 'kafka1:9092,kafka2:9092',  # List broker
    'group.id': 'my-consumer-group',                  # Consumer Group ID
    'auto.offset.reset': 'earliest',                  # Or 'latest' depending on use-case
    'enable.auto.commit': False,                      # Turn off auto commit for control manual
    'max.poll.interval.ms': 300000,                   # Maximum time between polls (5 minutes)
    'session.timeout.ms': 10000,                      # Waiting time before considering consumer dead
    'heartbeat.interval.ms': 3000,                    # Heartbeat frequency
    'fetch.wait.max.ms': 500,                         # Maximum wait time when fetching data
    'fetch.min.bytes': 1024 * 1024,                   # Fetch at least 1MB per fetch
    'fetch.max.bytes': 1024 * 1024 * 50,             # 50MB limit per fetch
    'max.partition.fetch.bytes': 1024 * 1024 * 10,   # max 10MB/partition
    'queued.max.messages.kbytes': 1024 * 1024,       # Message queue limit
    'partition.assignment.strategy': 'cooperative-sticky'  # Assign partition strategy
}
consumer = Consumer(config)
```

### Using kafka-python
```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'your-topic',
    bootstrap_servers='kafka1:9092,kafka2:9092',
    group_id='my-consumer-group',
    auto_offset_reset='earliest',        # 'earliest' or 'latest'
    enable_auto_commit=False,            # disbale auto commit
    max_poll_records=500,                # Maximum number of records per poll
    max_poll_interval_ms=300000,         # 5 minutes
    session_timeout_ms=10000,
    heartbeat_interval_ms=3000,
    fetch_max_bytes=52428800,            # 50MB
    fetch_min_bytes=1048576,             # 1MB
    fetch_max_wait_ms=500,
    max_partition_fetch_bytes=10485760,  # 10MB/partition
    partition_assignment_strategy=[
        'roundrobin', 'cooperative_sticky'  # or 'range'
    ]
)
```
## Best Practices when implementing Consumer
1. Efficient message handling
    - Batch Processing: Process messages in batches to reduce I/O.
    ```python
    while True:
        messages = consumer.poll(timeout_ms=1000, max_records=500)
        if not messages:
            continue
        process_batch(messages)  # batch processing
        consumer.commit()        # Commit offset after process
    ```

    - Parallel Processing: Use multi-thread or asyncio.
    ```python
    from concurrent.futures import ThreadPoolExecutor

    def process_message(msg):
        # Proces message here

    with ThreadPoolExecutor(max_workers=10) as executor:
        for msg in consumer:
            executor.submit(process_message, msg)
    ```
2. Offset Management
    - Manual Commit: Commit only when the message has been successfully processed.
    ```python
    try:
        for msg in consumer:
            process(msg)
            consumer.commit(msg)  # Commit message processed
    except Exception as e:
        handle_error(e)
    ```
3. Error Handling and Retry
    - Dead Letter Queue (DLQ): Push error messages to a separate topic.
    ```python
    from confluent_kafka import Producer
    
    dlq_producer = Producer({'bootstrap.servers': 'kafka:9092'})
    for msg in consumer:
        try:
            process(msg)
        except Exception as e:
            dlq_producer.produce('dead_letter_topic', msg.value())
            dlq_producer.flush()
    ```

    - Retry Policy: Perform retry with backoff.
    ```python
    import time

    def process_with_retry(msg, max_retries=3):
        for attempt in range(max_retries):
            try:
                process(msg)
                return
            except Exception:
                time.sleep(2 ** attempt)
        raise RetryFailedError()
    ```
### Optimizing Kafka system
1. Tuning Broker:
    - Increase `num.network.threads` and `num.io.threads`.
    - Set `log.retention.bytes` and `log.segment.bytes` accordingly.
2. Partitioning
    - Number of partitions = number of consumer instances * factor (usually 2-3x).
    - Use key-based partitioning if ordering is required.
3. Monitoring
    - Using Kafka Manager, Prometheus + Grafana.
    - Monitor consumer lag: kafka-consumer-groups --describe.
