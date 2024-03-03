# Practicing Kubernetes for Production
Build microservice containers with Docker, create CICD pipeline with GitHub Action, build and push images to Artifact Registry, and deploy to GKE!

## Steps to use
Most of the steps are aligned with this [Medium blog](https://medium.com/@gravish316/setup-ci-cd-using-github-actions-to-deploy-to-google-kubernetes-engine-ef465a482fd). Quite precise and informative! But there are some extra steps need to take to make it work for this repo

### 1. Create Service Account JSON credentials
After creating GKE cluster and Artifact Registry repo, follow the steps in the Medium blog to obtain the Service Account credentials in JSON file which will be used later.
### 2. Use local Gcloud to test the connection
If gcloud cli is not installed, follow [this official doc](https://cloud.google.com/sdk/docs/install-sdk).
```
$ gcloud auth activate-service-account --key-file=<path to the JSON key>
$ gcloud auth configure-docker <artifact registry region>-docker.pkg.dev
$ gcloud auth print-access-token | docker login -u oauth2accesstoken --password-stdin https://<artifact registry region>-docker.pkg.dev
```
After authentication passes, test it by pushing a local docker image to the registry
```
$ docker build -t <artifact registry region>-docker.pkg.dev/<project ID>/<artifact registry repository>/<image name>:<image tag> -f Dockerfile .
$ docker push <artifact registry region>-docker.pkg.dev/<project ID>/<artifact registry repository>/<image name>:<image tag>
```
Check the GCP console that the image is there.
### 3, Create a secret inside the GKE
This repo requires 1 secret, named `pgpassword` to store the password of PostgreSQL. Open the GKE shell and run the followings
```
$ gcloud config set project <PROJECT_ID>
$ gcloud config set compute/zone <CLUSTER_REGION>
$ gcloud container clusters get-credentials <CLUSTER_NAME>
```
Now the `kubectl` will be working in the context of our GKE cluster. Create secrets with the following comannds:
```
$ kubectl create secret generic pgpassword --from-literal PGPASSWORD=123456789
```
Go to the GKE ui and check out the tab "Secrets" to verify the creation
### 4. Install and setup Helm
Since we use one 3rd-party object i.e. `ingress-nginx`, we will use helm to manage this package. Inside the GKE shell, run the following to install Helm:
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
Install ingress-nginx with Helm with the following command:
```
$ helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
Verify the installation:
```
$ kubectl get service ingress-nginx-controller --namespace=ingress-nginx
```
Now all things are setup and ready!

**Note:** For the local development if you don't have Helm installed, you can run the following command to install ingress-nginx: `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml` (ref: [official doc](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start))
### 5. Create secrets for GitHub actions
Click `Settings` in the repo page > `Security/Secrets and variables` on the right handside > `Actions` > create new repo secrets (for this repo, we need 2 of them: `GKE_SA_KEY` and `GKE_PROJECT`)
### 6. Push to the master branch!
Push to the master branch to trigger the GitHub action
  
**Note:** variables in GitHub actions can be found in this [official doc](https://docs.github.com/en/actions/learn-github-actions/contexts)
### 7. Open the public link
On GKE ui, on the right handside, under `Networking` section, select `Gateways, Services & Ingress`, look for `ingress-nginx-controller` with the type of `External Load Balancer`, you'll find the endpoints there and that's your running public url!