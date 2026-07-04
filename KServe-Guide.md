Practical 6: Model Serving using KServe
Step 1: Create a Kubernetes Cluster
Create a local Kubernetes cluster using KIND.
kind create cluster --name mlops-kserve-demo

Verify the current Kubernetes context.
kubectl config current-context
kubectl config use-context kind-production-mlflow-setup  
(To use a cluster)

Step 2: Install Cert Manager
KServe uses admission webhooks secured with TLS certificates. Install Cert Manager before installing KServe.
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

Verify the installation.
kubectl get pods -n cert-manager

All pods should be in the Running state.
Step 3: Install KServe
Create the KServe namespace.
kubectl create namespace kserve

Install the KServe CRDs.
helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.16.0 \
  -n kserve \
  --wait

Install the KServe Controller.
helm install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.16.0 \
  -n kserve \
  --set kserve.controller.deploymentMode=RawDeployment \
  --wait

Verify that the controller is running.
kubectl get pods -n kserve

Expected output:
NAME                                         READY   STATUS
kserve-controller-manager-xxxxxxxxxx         2/2     Running
Step 4: Create a Namespace for Model Deployment
kubectl create namespace ml
Verify:
kubectl get ns
Step 5: Create the InferenceService Manifest
Create a file named intent-model.yaml.
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService

metadata:
  name: intent-model

spec:
  predictor:
    model:
      modelFormat:
        name: sklearn

      storageUri: Valid-URI(Recommended is to push your model to a folder in S3 and copy the uri of the FOLDER)

      resources:
        requests:
          cpu: "100m"
          memory: "512Mi"

        limits:
          cpu: "1"
          memory: "1Gi"
Step 6: Create an AWS Secret
Create a Kubernetes Secret containing your AWS credentials.
kubectl create secret generic aws-secret \
-n ml \
--from-literal=AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY> \
--from-literal=AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_ACCESS_KEY>
Step 7: Configure the Service Account
Annotate the default ServiceAccount with the S3 endpoint and AWS region.
kubectl annotate serviceaccount default \
-n ml \
serving.kserve.io/s3-endpoint=s3.amazonaws.com \
serving.kserve.io/s3-region=eu-north-1

Attach the AWS Secret.
kubectl patch serviceaccount default \
-n ml \
-p '{"secrets":[{"name":"aws-secret"}]}'
Step 8: Deploy the Model
Apply the InferenceService manifest.
kubectl apply -f intent-model.yaml -n ml

Expected output:
inferenceservice.serving.kserve.io/intent-model created
Step 9: Verify the Deployment
Watch the InferenceService.
kubectl get isvc -n ml -w
Expected output:
NAME           URL                                 READY
intent-model   http://intent-model-ml.example.com  True

Watch the Pods.
kubectl get pods -n ml -w
Expected output:
NAME                                    READY   STATUS
intent-model-predictor-xxxxx            1/1     Running
Describe the InferenceService.
kubectl describe isvc intent-model -n ml
Step 10: Troubleshooting
If the model does not become READY, inspect the logs.
Storage Initializer logs:
kubectl logs <pod-name> -n ml -c storage-initializer
Predictor logs:
kubectl logs <pod-name> -n ml
Useful commands:
kubectl describe pod <pod-name> -n ml
kubectl get events -n ml --sort-by=.lastTimestamp
Step 11: Expose the Service
Forward the Predictor Service to the local machine.
kubectl port-forward -n ml svc/intent-model-predictor 8089:80

Keep this terminal running.
Step 12: Test the Model
Send an inference request using the KServe V1 REST API.
curl -X POST http://localhost:8089/v1/models/intent-model:predict \
-H "Content-Type: application/json" \
-d '{"instances":["I want to cancel my subscription"]}'

Expected response:
{
  "predictions": [
    "complaint"
  ]
}


Architecture
                +--------------------+
                 |  Amazon S3 Bucket  |
                 |     model.pkl      |
                 +---------+----------+
                           |
                           | storageUri
                           |
                 +---------v----------+
                 | Storage Initializer|
                 +---------+----------+
                           |
                           |
                 +---------v----------+
                 | KServe Predictor   |
                 | sklearnserver      |
                 +---------+----------+
                           |
                     REST API (HTTP)
                           |
                 +---------v----------+
                 |      curl Client   |
                 +--------------------+
