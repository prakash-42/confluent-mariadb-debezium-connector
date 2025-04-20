# Introduction

This repository provides the [debezium-connector-mariadb.zip](./debezium-connector-mariadb.zip) file, which is a modified version of the official Debezium MariaDB connector (version 3.0.6). This modification makes the connector compatible with a self-hosted Confluent Platform running on Kubernetes.

**WARNING**: While this method should work with most Connect workers and Debezium connectors, it has only been tested with the following combination of versions:

| Component                          | Version | Description                                                                 |
|------------------------------------|---------|-----------------------------------------------------------------------------|
| `debezium-mariadb-connector`       | 3.0.6   | The Debezium connector being used.                                          |
| `confluentinc/cp-server-connect`   | 7.9.0   | Docker image for the Connect worker that runs the connector.               |
| `confluentinc/confluent-init-container` | 2.7.0   | Docker image for the init container that downloads and installs the connector ZIP. |

For details on how this connector was created, refer to the later sections of this file.

---

# Why Use This Connector?

The self-hosted Confluent Platform only supports connectors available on [Confluent Hub](https://www.confluent.io/hub/). However, as of April 2025, the [official Debezium MariaDB connector](https://debezium.io/documentation/reference/stable/connectors/mariadb.html) is not available on Confluent Hub. 

While the Confluent Platform allows running external connectors by providing a connector ZIP file, the official Debezium connectors are not compatible with Confluent Connect workers running on Kubernetes. Additionally, Confluent Hub often lags behind the latest open-source connector versions, making it difficult to use the most up-to-date connectors.

The method described in this repository can also be applied to other official Debezium connectors to make them compatible with the Confluent Platform.

---

# How to Run?

To run the [example.yml](./example.yml) file provided in this project, you need to install the Confluent Kubernetes Operator. Follow the steps outlined in the [Confluent Kubernetes Operator documentation](https://docs.confluent.io/operator/current/overview.html). Below is a quick summary of the steps to get started:

```bash
# Add the Helm repository for Confluent
helm repo add confluentinc https://packages.confluent.io/helm

# Install the Confluent Kubernetes Operator
helm upgrade --install operator confluentinc/confluent-for-kubernetes -n <namespace>

# Create secrets for Confluent Kafka and Schema Registry
kubectl create secret generic ccloud-credentials --from-file=plain.txt=./ccloud-credentials.txt -n <namespace>
kubectl create secret generic ccloud-sr-credentials --from-file=basic.txt=./ccloud-sr-credentials.txt -n <namespace>

# Install the connect worker using example.yml
kubectl apply -f example.yml -n <namespace>
```

# How Was This Connector Created?
For details on how this version of the connector was created, refer to the [internal-details.md](./internal-details.md) file. This information is useful for debugging installation issues or creating connectors for different versions of the Debezium connector.
