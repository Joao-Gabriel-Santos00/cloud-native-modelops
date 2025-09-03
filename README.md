# Cloud-Native ModelOps Platform

This project contains the Kubernetes manifests and Helm chart for deploying a complete, end-to-end MLOps platform. It takes the services from the [ModelOps Studio](https://github.com/Joao-Gabriel-Santos00/modelops-studio) project and packages them for a production-style, cloud-native environment. The final deliverable is a single Helm chart that can deploy the entire resilient and observable stack with one command.

## Why I Built This Project

I built this project as the next evolution of my MLOps Studio platform to demonstrate my skills in deploying complex, multi-service applications onto a production-grade orchestration platform. The core challenge was to move beyond a local `docker-compose` setup and re-architect the entire platform for Kubernetes, the industry standard for container orchestration.

By packaging the system into a single, reusable Helm chart and implementing production-readiness features like health checks and resource management, this project serves as tangible proof of my ability to manage the full lifecycle of an ML system—from local development to scalable, cloud-native deployment.

## Architecture

The entire platform runs within a local Kubernetes cluster. The services are designed to be decoupled and communicate over the internal Kubernetes network. State is persisted using Kubernetes Persistent Volumes, ensuring data for services like MinIO survives pod restarts and redeployments.

![Architecture Diagram](docs/architecture.png)
*(Note: You will need to create this diagram using a tool like Excalidraw or Diagrams.net and place it in a `docs` folder)*

## Key Features & Concepts Demonstrated

*   **Container Orchestration:** Deployed a multi-service, stateful application on Kubernetes, managing the full lifecycle of each component with declarative manifests.
*   **Infrastructure as Code (IaC):** Packaged the entire platform into a reusable and configurable **Helm chart**, enabling one-command, version-controlled deployments.
*   **Stateful Service Management:** Used `PersistentVolumeClaims` to provide stable, persistent storage for the MinIO object store, ensuring model artifacts are not lost.
*   **Configuration Management:** Managed all application configuration (image tags, ports, resources, replica counts) centrally in a single `values.yaml` file for easy and safe updates.
*   **Production Readiness:** Implemented **Liveness and Readiness probes** for all services, enabling Kubernetes to perform automated health checks and self-healing by restarting unhealthy pods.
*   **Resource Management:** Defined CPU and memory **requests and limits** for all components, ensuring cluster stability and preventing resource contention.

## Getting Started

Follow these instructions to deploy the entire MLOps stack on your local machine.

### Prerequisites

*   **Docker Desktop:** with the Kubernetes engine enabled.
*   **kubectl:** command-line tool for interacting with Kubernetes (included with Docker Desktop).
*   **Helm v3:** the package manager for Kubernetes.

### Deployment Steps

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/your-username/cloud-native-modelops.git
    cd cloud-native-modelops
    ```

2.  **Update Configuration (IMPORTANT):** Before deploying, open `modelops-stack/values.yaml` and update the `modelServer.image.repository` value to point to your Docker Hub username.
    ```yaml
    # in modelops-stack/values.yaml
    modelServer:
      image:
        repository: your-username/model-server # <-- CHANGE THIS
        tag: "0.3" # Or your latest working tag
    ```

3.  **Deploy the MinIO Secret:** This secret is referenced by the Helm chart but managed separately.
    ```powershell
    kubectl apply -f kubernetes/minio-secret.yml
    ```

4.  **Install the Helm Chart:** This single command will deploy all services, deployments, and configurations.
    ```powershell
    helm install modelops ./modelops-stack/
    ```

5.  **Verify the Deployment:** Wait a minute or two for all containers to pull and start. You can watch the progress with:
    ```powershell
    kubectl get pods -w
    ```
    Once all pods show `STATUS` as `Running`, the platform is ready.

## Access Points & Usage

The services are exposed on your local machine via `NodePort` services.

| Service          | Address                   | Credentials             |
| :--------------- | :------------------------ | :---------------------- |
| MLflow UI        | `http://localhost:30082`  | N/A                     |
| MinIO Console    | `http://localhost:30083`  | `minioadmin`/`minioadmin` |
| Model Server API | `http://localhost:30081/docs` | N/A                     |

### End-to-End Test Flow

1.  **Create MinIO Bucket:** Navigate to the MinIO Console, go to "Buckets," and create a new bucket named `mlflow`.
2.  **Run Training Job:** Execute the following PowerShell command to run a training job, which registers a model in MLflow and stores the artifact in MinIO. (Remember to use your trainer image name).
    ```powershell
    kubectl run model-trainer `
      --image=your-username/trainer:latest `
      --env="MLFLOW_TRACKING_URI=http://mlflow-service:5000" `
      --env="MLFLOW_S3_ENDPOINT_URL=http://minio-service:9000" `
      --env="AWS_ACCESS_KEY_ID=minioadmin" `
      --env="AWS_SECRET_ACCESS_KEY=minioadmin" `
      --rm -it --restart=Never --command -- python services/trainer/train.py --model-name ModelOpsStudioModel
    ```
3.  **Deploy Model:** Go to the MLflow UI, find the `run_id` of the new run, and use the `/deploy` endpoint in the FastAPI UI to load the model.
4.  **Get Prediction:** Use the `/predict` endpoint with a valid payload (a list of 30 numbers) to get a real-time prediction.

## Configuration

The entire stack is configured via the `modelops-stack/values.yaml` file

## Next Steps & Future Work

This project successfully completes the "Scalability Proof" groundwork from the original ModelOps Studio. To further advance this toward a true enterprise-grade system, the following areas would be tackled next:

*   **GitOps for CI/CD:** Integrate a GitOps tool like ArgoCD or FluxCD. This would create a fully automated CI/CD pipeline where merging a change to the Helm chart in Git automatically triggers a deployment to the Kubernetes cluster.

*   **Automated Training Pipelines:** Integrate a workflow orchestrator like Apache Airflow (running on Kubernetes) to replace the manual kubectl run command for training. This would enable scheduled retraining or trigger-based retraining (e.g., on data drift).

*   **True Cloud Deployment:** Deploy this Helm chart to a managed Kubernetes service (like AWS EKS, GKE, or AKS) and replace the MinIO NodePort with a cloud-native Ingress controller and the minio-secret with a managed secret store (like AWS Secrets Manager).
  