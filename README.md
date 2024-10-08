# Kubernetes Deployment with KIND and Jenkins using Helm

## Overview

This guide walks you through the process of setting up a Kubernetes cluster using KIND (Kubernetes IN Docker), and then deploying Jenkins using a custom Helm chart.

## Prerequisites

Before you start, make sure you have the following installed:

- Docker: [Installation Guide](https://docs.docker.com/get-docker/)
- KIND: [Installation Guide](https://kind.sigs.k8s.io/docs/user/quick-start/)
- Helm: [Installation Guide](https://helm.sh/docs/intro/install/)

## Step 1: Install KIND

1. **Install KIND**

   Follow the instructions for your operating system:

   - **Linux**:

    ```sh
    # For AMD64 / x86_64
    [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64

    # For ARM64
    [ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-arm64

    chmod +x ./kind

    sudo mv ./kind /usr/local/bin/kind
    ```

2. **Verify Installation**

   Check the KIND version to ensure it’s installed correctly:

   ```sh
   kind version
   ```

## Step 2: Create a KIND Cluster

1. **KIND Configuration File**

   There is a configuration file called `kind-config.yaml`:

   ```yaml
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
     - role: control-plane
       kubeadmConfigPatches:
       - |
         kind: InitConfiguration
         nodeRegistration:
           kubeletExtraArgs:
             node-labels: "ingress-ready=true"
       extraPortMappings:
       - containerPort: 80
         hostPort: 80
         protocol: TCP
       - containerPort: 443
         hostPort: 443
         protocol: TCP
     - role: worker
   ```

2. **Create the Cluster**

   Run the following command to create the cluster:

   ```sh
   kind create cluster --config kind-config.yaml
   ```

3. **Verify the Cluster**

   Check that the cluster is up and running:

   ```sh
   kubectl cluster-info --context kind-kind
   ```

   You should see information about your cluster's control-plane and other components.

## Step 3: Install the NGINX Ingress Controller

Before deploying Jenkins, you need to install the NGINX Ingress Controller to handle ingress traffic:

1. **Install NGINX Ingress Controller**

   Run the following command:

   ```sh
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
   ```

   This will deploy the NGINX Ingress Controller into your KIND cluster, which will manage incoming HTTP(S) requests and route them to the appropriate services.

2. **Verify the Ingress Installation**

   Check that the ingress controller is running:

   ```sh
   kubectl get pods -n ingress-nginx
   ```

   You should see the ingress controller pods running.

## Step 4: Configure Jenkins for Kubernetes

### Create a Kubernetes Cloud in Jenkins

Since Jenkins is running inside the Kubernetes cluster, we can use the Kubernetes plugin to dynamically provision agents (pods) within the cluster.

1. **Access Jenkins Configuration**:
   - Go to the Jenkins dashboard.
   - Navigate to "Manage Jenkins" > "Manage Nodes and Clouds" > "Configure Clouds".

2. **Add a New Kubernetes Cloud**:
   - Click on "Add a new cloud" and select "Kubernetes".
   - Name the cloud (e.g., `kubernetes`).

3. **Configure Kubernetes Cloud**:
   - **Kubernetes URL**: Leave this field blank since Jenkins is running inside the cluster and can auto-discover the API server.
   - **Kubernetes Namespace**: Specify the namespace where Jenkins is running (e.g., `default`).
   - **Jenkins tunnel**: Leave this field blank (auto-configured).

4. **Enable WebSockets**:
   - In the Kubernetes Cloud configuration, scroll down and enable the option "Use WebSocket" under the "Advanced" section. This is crucial for Jenkins to communicate effectively with agents.

5. **Test Connection**:
   - Click "Test Connection" to ensure Jenkins can connect to the Kubernetes API.

### Step 5: Deploy Jenkins with Helm

1. **Navigate to the Helm Chart Directory**

   Change to the directory where the Jenkins Helm chart is located:

   ```sh
   cd path/to/jenkins
   ```

2. **Install Jenkins**

   Deploy Jenkins using Helm with your custom values file:

   ```sh
   helm install jenkins . --values my-values.yaml
   ```

   This command deploys Jenkins using the Helm chart in the current directory and applies the configuration specified in `my-values.yaml`.

3. **Verify the Deployment**

   Check the status of the Jenkins deployment:

   ```sh
   kubectl get pods
   ```

   You should see Jenkins pods running. For example:

   ```
   NAME                        READY   STATUS    RESTARTS   AGE
   jenkins-xxxxxx-yyyyy        1/1     Running   0          5m
   ```

## Cleanup

To remove the Jenkins deployment and KIND cluster:

1. **Delete Jenkins**

   ```sh
   helm uninstall jenkins
   ```

2. **Delete KIND Cluster**

   ```sh
   kind delete cluster
   ```