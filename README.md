## 1. Introduction

The [Clever Operator](https://github.com/CleverCloud/clever-operator) is an open-source project designed to seamlessly integrate [Clever Cloud](https://www.clever-cloud.com/)’s managed services into Kubernetes environments. By leveraging Kubernetes [Custom Resource Definitions (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)), the Clever Operator enables developers to manage Clever Cloud resources directly from their Kubernetes clusters, aligning cloud-native practices with Clever Cloud’s powerful platform.

Modern applications often require a combination of containerized workloads and managed services, such as databases or caches. Managing these resources separately across platforms can become complex and error-prone. The Clever Operator simplifies this process by acting as a bridge, allowing developers to define and interact with Clever Cloud’s resources using familiar Kubernetes paradigms.

Key features of the Clever Operator include:

- **Custom Resource Definitions (CRDs):** Extend Kubernetes capabilities to manage Clever Cloud services like PostgreSQL, Redis, and more.
- **Declarative Resource Management:** Use YAML manifests to declare and maintain the desired state of your services.
- **Seamless Integration:** Interact with Clever Cloud’s API securely and efficiently.
- **Scalability and Flexibility:** Manage resources across multiple namespaces with consistent configurations.

This documentation will guide you through:

1. Installing and configuring the Clever Operator in your Kubernetes cluster.
2. Managing Clever Cloud resources such as PostgreSQL and Redis through examples.
3. Troubleshooting and monitoring the operator for efficient operations.

Whether you are deploying your first application or managing complex, multi-environment systems, the Clever Operator empowers you to streamline operations and focus on building impactful software.

## 2. Prerequisites

Before you begin, ensure that you have the following tools and resources based on your intended actions:

#### To Build the Operator

- **Git:** Clone the Clever Operator repository to access the source code.
- **Rust Toolchain:** Install the Rust programming language and its toolchain to compile the operator from source. Follow the installation guide at [https://rustup.rs/](https://rustup.rs/).
- **Docker:** Build container images for deploying the operator in Kubernetes.

#### To Deploy the Operator

- **Kubernetes Cluster:** Ensure you have access to a running Kubernetes cluster.
- **Kubectl:** Install Kubernetes command-line tool for managing cluster resources Installation guide available at https://kubernetes.io/docs/tasks/tools/.
- **Clever Cloud Account Credentials:** Obtain API tokens and secrets from your Clever Cloud account to configure the operator.

These prerequisites are essential for getting started with the Clever Operator, whether you're contributing to its development or deploying it in production.

## **3. Installation**

We suggest you to deploy the Clever Operator either directly from Dockerhub or using the Helm chart.

### 3.1. Deploying from DockerHub

Applying the deployment scripts:

```bash
kubectl apply -f https://raw.githubusercontent.com/CleverCloud/clever-operator/main/deployments/kubernetes/v1.24.0/10-custom-resource-definition.yaml 
kubectl apply -f https://raw.githubusercontent.com/CleverCloud/clever-operator/main/deployments/kubernetes/v1.24.0/20-deployment.yaml
```


### 3.2. Installing via Helm Chart

1. Configuring `values.yaml` in `deployments/kubernetes/helm` with your values.

2. Installing the chart:
	```bash
helm install clever-operator -n clever-operator --create-namespace -f values.yaml .
```


### 3.3. Building from Source

1. Cloning the repository:

```bash
git clone https://github.com/CleverCloud/clever-operator.git
cd clever-operator
```

2. Building the binary:

```bash
make build
```

 3. Running the operator:

```bash
target/release/clever-operator
```

### 3.4. Building and Deploying the Docker Image

1. Building the Docker image:

	```bash
	DOCKER_IMG=<your-registry>/<your-namespace>/clever-operator:latest make docker-build
	```

1. Pushing the image to your registry:

	```bash
	DOCKER_IMG=<your-registry>/<your-namespace>/clever-operator:latest make docker-push
	```

3. Updating the Kubernetes deployment script: 
	
	Modify `deployments/kubernetes/v1.24.0/20-deployment.yaml` to use your Docker image.

4. Deploying to Kubernetes:
	```bash
	make deploy-kubernetes
	```


## 4. Configuration

The Clever Operator requires configuration to connect to Clever Cloud's API and manage resources within your Kubernetes cluster. Configuration options are available at two levels: global (applies to all namespaces) and namespace-specific.

For details on how to obtain these credentials, follow the instructions on the [How to obtain the credentials for the Clever Operator](./credentials.md) document.

### 4.1 Global Configuration

Global configuration settings apply across all namespaces and are defined via environment variables or configuration files.

- **Environment Variables:**
    
    - `CLEVER_OPERATOR_API_ENDPOINT`: The endpoint for the Clever Cloud API.
    - `CLEVER_OPERATOR_API_TOKEN`: Your Clever Cloud API token.
    - `CLEVER_OPERATOR_API_SECRET`: The secret associated with your API token.
    - `CLEVER_OPERATOR_API_CONSUMER_KEY`: Your Clever Cloud consumer key
    - `CLEVER_OPERATOR_API_CONSUMER_SECRET`: Your Clever Cloud consumer secret.

- **Configuration Files:** By default, if the `--config` flag is not provided to the binary, the operator will look at the following locations to retrieve its configuration (in order of priority):
    
    1. `/usr/share/clever-operator/config.{toml,yaml,json}`
    2. `/etc/clever-operator/config.{toml,yaml,json}`
    3. `$HOME/.config/clever-operator/config.{toml,yaml,json}`
    4. `$HOME/.local/share/clever-operator/config.{toml,yaml,json}`
    5. `config.{toml,yaml,json}` (in the current working directory)


### 4.2 Namespace-Level Configuration

Namespace-level configurations override the global settings for specific namespaces. They are defined using a Kubernetes Secret resource named `clever-operator` with the `config` key.

- **Creating a Namespace-Level Configuration:** Create a Kubernetes Secret with the necessary configuration keys:
    
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: clever-operator
      namespace: my-namespace
    stringData:
      config: |-
        api:
          endpoint: "https://api.clever-cloud.com/v2"
          token: "<your-api-token>"
          secret: "<your-api-secret>"
          consumerKey: "<your-consumer-key>"
          consumerSecret: "<your-consumer-secret>"
    ```
    
- **Applying the Configuration:** Apply the Secret to your namespace:
    
    ```bash
    kubectl apply -f namespace-config.yaml
    ```
    

The operator automatically detects and applies namespace-specific configurations when interacting with resources in that namespace.

### 4.3 Validating Configuration

To ensure your configuration is applied correctly, check the operator logs for any errors or warnings:

```bash
kubectl logs -n clever-operator-system <operator-pod-name>
```

## 5. Usage Examples

The Clever Operator enables you to manage Clever Cloud resources directly from your Kubernetes cluster using custom resources. Below are examples for PostgreSQL and Redis.

### 5.1 Managing PostgreSQL Resources

- **Creating a PostgreSQL Instance:** Define a YAML manifest for the PostgreSQL resource:
    
    ```yaml
  apiVersion: api.clever-cloud.com/v1
  kind: PostgreSql
  metadata:
    name: clever-fest
    namespace: default
  spec:
    organisation: "<your-organisation>"
    options:
      version: 14
      encryption: false
    instance:
      plan: "dev"
      region: "par"
    ```
    
    Apply the manifest to your cluster:
    
    ```bash
    kubectl apply -f postgresql.yaml
    ```
    
- **Verifying the Deployment:** Check the status of the PostgreSQL resource:
    
    ```bash
    kubectl get postgresql my-postgresql -o yaml
    ```
    
- **Accessing PostgreSQL:** Retrieve the connection details from the Clever Cloud dashboard or the resource’s status field.


#### 5.2 Managing Redis Resources

- **Creating a Redis Instance:** Define a YAML manifest for the Redis resource:
    
    ```yaml
    apiVersion: clever-cloud.com/v1
    kind: Redis
    metadata:
      name: my-redis
      namespace: default
    spec:
      organisation: "<your-organisation>
      options:
        version: 626
        encryption: false
      instance:
        region: par
        plan: s_mono
    ```
    
    Apply the manifest to your cluster:
    
    ```bash
    kubectl apply -f redis.yaml
    ```
    
- **Verifying the Deployment:** Check the status of the Redis resource:
    
    ```bash
    kubectl get redis my-redis -o yaml
    ```
    
- **Accessing Redis:** Retrieve the connection details from the Clever Cloud dashboard or the resource’s status field.

These examples demonstrate the simplicity and power of using the Clever Operator to manage cloud resources declaratively in Kubernetes.

