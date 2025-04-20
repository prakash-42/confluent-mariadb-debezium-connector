# Introduction

This repository provides a [debezium-connector-mariadb.zip](./debezium-connector-mariadb.zip) file that is a modified version of the official debezium mariadb connector (3.0.6), so that it becomes compatible with self-hosted confluent platform running on kubernetes.

**WARNING**: While this method should work with most connect workers and debezium connectors, we have only tested this with the following combination of versions:
|Component | Version | Comment|
| ------------- | ------------- | ------------- |
|debezium-mariadb-connector| 3.0.6| This is the debezium connector we're trying to run |
|confluentinc/cp-server-connect| 7.9.0| Docker image for connect worker that runs the connector |
|confluentinc/confluent-init-container| 2.7.0| Docker image for init container that downloads and installs the external connector zip |

You can read details of how this connector was created in the later sections of this file.

# Why this connector?

Confluent platform (self-hosted) only supports connectors available on [confluent hub](https://www.confluent.io/hub/), and it doesn't yet support the [official debezium mariadb connector](https://debezium.io/documentation/reference/stable/connectors/mariadb.html) (as of April 2025). Confluent platform also has option to run external connectors by providing the connector zip, but the official debezium connectors aren't compatible with the confluent connect workers running in kubernetes.

Also, confluent hub doesn't always have all the official versions of connectors, and are often a few versions behind the latest open-source connector. The method applied to create this connector should also be applicable to other official debezium connectors, to make them compatible with confluent platform.

# How to run?
To run the [example.yml](./example.yml) metioned in this project, you need to install the confluent kubernetes operator following steps at https://docs.confluent.io/operator/current/overview.html.
Here's a short list of steps to get you started:

```bash
# Add helm repo for confluent
helm repo add confluentinc https://packages.confluent.io/helm

# Install confluent kubernetes operator
helm upgrade --install operator confluentinc/confluent-for-kubernetes -n <namespace>

# Create secrets for confluent kafka and schema registry
kubectl create secret generic ccloud-credentials --from-file=plain.txt=./ccloud-credentials.txt -n  -n <namespace>
kubectl create secret generic ccloud-sr-credentials --from-file=basic.txt=./ccloud-sr-credentials.txt -n  -n <namespace>

# Install the connect worker from example.yml
kubectl apply -f example.yml -n <namespace>
```

# How was this created?
To get details on how this version of connector was created, check this file: [internal-details.md](./internal-details.md). This information would be helpful for debugging any installation issues, and for creating connector for different versions of the debezium connector.
