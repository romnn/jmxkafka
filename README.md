## Pre configured JMX kafka docker container

```
docker pull romnn/jmxkafka
```

This custom kafka container is based on [wurstmeister/kafka](https://hub.docker.com/r/wurstmeister/kafka), but also exposed a JMX metrics endpoint on **port 7072**.

Make sure to set the environment variable `KAFKA_OPTS: -javaagent:/usr/app/jmx_prometheus_javaagent.jar=7072:/usr/app/prom-jmx-agent-config.yml`.

#### Usage example with prometheus

Use this container instead of your normal kafka container in your `docker-compose.yml` configuration:

```yaml
prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.config.yml:/etc/prometheus/prometheus.yml
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention=12h
    ports:
      - ${CEPTA_PROMETHEUS_PORT}:9090

zookeeper:
    image: bitnami/zookeeper:latest
    expose:
      - 2181
    environment:
      ALLOW_ANONYMOUS_LOGIN: "yes"

kafka:
    image: romnn/jmxkafka:master
    expose:
      - 9092
      - 29092
      - 7072
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_ZOOKEEPER_CONNECT:  zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: INSIDE://kafka:9092,OUTSIDE://localhost:29092
      KAFKA_LISTENERS: INSIDE://:9092,OUTSIDE://:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INSIDE
      KAFKA_OPTS: -javaagent:/usr/app/jmx_prometheus_javaagent.jar=7072:/usr/app/prom-jmx-agent-config.yml
    depends_on:
      - zookeeper
```

Now you can configure your kafka broker as a datasource in your `prometheus.config.yml`:

```yaml
global:
  scrape_interval: 5s
  evaluation_interval: 5s
  external_labels:
    monitor: "my-monitor"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  # JMX exporter that runs on each kafka broker
  - job_name: "kafka"
    static_configs:
      - targets: ["kafka:7072"]
        labels:
          alias: "kafka"
```

#### Regards on scalability

Scalability with multiple sharded kafka brokers is not considered and might require a different configuration approach.
Contributions and discussion is greatly appreciated!