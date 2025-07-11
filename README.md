# Kubernetes Workshop

Kubernetes workshop for developers.

## Task 0

### Install the Azure cli

Windows

```ps
winget install -e --id Microsoft.AzureCLI
```

MacOS

```bash
brew update && brew install azure-cli
```

Linux

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

There are step-by-step installation instructions if you prefer not to run the script from Microsoft

### Log in to the correct subscription

We will be using a tenant with ID: `fdf88c80-36e9-45ee-a0b2-d7eb687e39eb`

```bash
az login --tenant fdf88c80-36e9-45ee-a0b2-d7eb687e39eb
```

### Install kubectl

```bash
az aks install-cli
```

### Set kubectl context

```bash
az aks get-credentials --name Sommerjobb_2025_k8s --resource-group Sommerjobb_BN_2025_rg
```

### Install k9s (Kubernetes GUI)

Windows

```ps
winget install k9s
```

MacOS

```bash
brew install derailed/k9s/k9s
```

Linux

Use package manager of choice or build from source: [install k9s](https://k9scli.io/topics/install/)

### How to deploy a namespace

In Kubernetes, namespaces provide a mechanism for isolating groups of resources within a single cluster. Names of resources need to be unique within a namespace, but not across namespaces.

- Update the file `manifest/namespace.yaml` with a unique name for your namespace, for instance your slack username
- Deploy the namespace by running: `kubectl apply -f namespace.yaml`

For our purposes we're going to be using the provided `husbanken` namespace.

## Task 1

You will now build an image and deploy it to the Azure Container Registry (ACR).
All containers must have a unique name within the ACR.

During this workshop you should choose a unique prefix so all your images gets a uniqe name, for instance your full name. Only use lowercase letters. Use this prefix instead of `<uri-prefix>` in the commands below.

### Build and push image to registry

- Open `aspnetapp\src\Program.cs` and update line 24 with a unique path
- Navigate to `aspnetapp` and build the Dockerfile:

#### Alternative 1: Build image in the cloud using ACR Tasks

```bash
az acr build --image <uri-prefix>/aspnet:v1 --registry sommerjobbbn2025acr --file Dockerfile .
```

#### Alternative 2: Build locally using podman

```bash
az acr login --name sommerjobbbn2025acr

podman build --tag sommerjobbbn2025acr.azurecr.io/<uri-prefix>/aspnet:v1 --file Dockerfile .

podman push sommerjobbbn2025acr.azurecr.io/<uri-prefix>/aspnet:v1
```

### Update task-1.yaml

- Update namespace to your own
- In `Pod` definition at `spec.containers[0].image` update to the correct uri for your image
- In `Ingress` definition at `spec.rules[0].http.paths[0].path` set the base path chosen in `Program.cs`

### Deploy in Kubernetes

- Run:

```bash
kubectl apply -f task-1.yaml
```

### Check that the service is available

- Find the IP address using k9s: `:ingress`
- Navigate to `ip-address/subdirectory` in the browser
  - The subdirectory is the path from `Program.cs`

## Task 2

You will now deploy a Svelte front end, which has nothing to do with the app you deployed in task 1.

### Build and push image to registry

- Navigate to `frontend\svelte.config.js` and update line 16 to a path where you will host the frontend.
  - This path cannot be the same path as the one you set in `aspnetapp`.
- Build the frontend image

```bash
az acr build --image <uri-prefix>/<name>:<tag> --registry sommerjobbbn2025acr --file Dockerfile .
```

### Update frontend.yaml with your own

- namespace
- subdirectory (base path)
- image prefix and name

### Deploy the frontend to Kubernetes

```bash
kubectl apply -f frontend.yaml
```

## Task 3

### Build and push image to registry

- Navigate to `database` and build and push the Dockerfile
- After updating all blanked out fields with appropriate values you can deploy `database.yaml`
- Check the pod with k9s. You should find the pod in a crash loop back off

### Add environment variables

- Check the `templates` folder to see an example of how one adds environment variables to a deployment
- Add these variables: `POSTGRES_PASSWORD:ඞඞඞ` `POSTGRES_DB: todo` `PG_HOST: postgres`
- Re-deploy and observe that the pod now starts

### Improve the deployment

In this step we will improve the deployment and get closer to best practices. The two big problems with this deployment are that data is being stored in the pod's local storage, which means that if the pod is deleted all the data is lost. Obviously not ideal for a database.

The other large issue is that the 'secret' password is in clear text in the yaml file which is committed to version control.

Finally the environment variables are defined directly on the deployment meaning we have to redeploy each time we want to update the config, and if multiple service share some config it has to be duplicated in all services.

We will fix the last problem first.

### Move config to a config map

- Create a file where you can define your config map. Theres a template to get you started in the template folder
- Remove the env section from the database deployment and add the same keys to the config map
- Deploy the config map
- Add an envFrom section to your database deployment, see templates for help
- Redeploy the database
- Add the same envFrom section to your frontend and it should now be able to connect to the database

### Use a Persistent Volume for the database data

To externalize the storage we need to ask kubernetes for a storage volume using a Persistent Volume Claim. Then we need to configure our database to mount the volume and store data there.

- Add a `PersistentVolumeClaim` to `database.yaml` See the template file for help
- Add the volumes section
- Add the volumeMounts section
- Update the config map with this entry: `PGDATA: /var/lib/postgresql/data/pgdata`
- Redploy the resources you have changed

Now you should be able to add todos from the frontend, delete the database with `Ctrl` + `k` in k9s, and see the same todos once the database restarts.

## Task 4

### Configure network policy

Default Kubernetes policy is to allow all traffic. This is not in line with the principle of least privilege. Therefore a good practice is to deploy a network policy which disallows all traffic, and explicitly allows only desired connections.

- Create and deploy a deny all network policy. See the template file `deny-all-policy.yaml` for help
- Check the frontend to see that it has lost connection to the database
- Deploy a network policy which only allows the required communication. See the template file
- Check the frontend again to see that communication is re-established

## Task 5

Optionally, install Kustomize: <https://kubectl.docs.kubernetes.io/installation/kustomize/>

It is also possible to use Kustomize via kubectl e.g. `kubectl kustomize <kustomization_directory>`

Use the demo springboot application as an example for creating kustomize files with overlays for the aspnetcore and/or the frontend application.

Deploy the application(s) to AKS in two different environments e.g. Prod and Test.
(since we're only working with a single namespace in the cluster, prefix the deployment with the environment)

Note: There's an extra README.md file in the there.
