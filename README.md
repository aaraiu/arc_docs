# Github Actions on Kubernetes

> [!TIP]
> Official documentation available at https://docs.github.com/en/actions

GitHub offers their own hosted virtual machines to run workflows, with an environment of tools and packages available for GitHub Actions to use. These are the Github provided runners, machines that execute jobs in a Actions workflow. 

As an alternative, there are self-hosted runners, which you can run and customize in your own infrastructure.

## Actions Runner Controller (ARC)

### Deploying ARC

The Actions Runner Controller, or short ARC, is a Kubernetes operator that orchestrates and scales self-hosted runners for your GitHub Actions.

ARC creates a few custom resources that automatically scale pods based on your workload. Here we can see a diagram of the main resources used.

An ARC deployment applies these resources onto a Kubernetes cluster. Once applied, it creates a set of Pods that contain your self-hosted runners' containers. With ARC, GitHub can treat these runner containers as self-hosted runners and allocate jobs to them as needed.

```shell
NAMESPACE="arc-systems"
helm install arc \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

After we install the chart we can see the `controller` pod running, as well as the deployment and a replica set.
```shell
❯ kubectl get pods -n arc-systems
NAME                                     READY   STATUS    RESTARTS   AGE
arc-gha-rs-controller-7fcd65b66b-2bxc6   1/1     Running   0          54s
```

## Deploying runner scale set

Now that we have the controller set up, we should generate a Personal Access token or [authenticate with a GitHub app](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api#authenticating-arc-with-a-github-app). In our example we use, the preferred way, a Github App key and configure a runner scale set. It is recommended to provision the scale set in a different namespace and keep the GitHub credentials as Kubernetes secrets and just pass it’s reference.

Here we have the command to create the kubernetes secret with the App ID, Installation ID and the private key:
```shell
❯ kubectl create secret generic gha-auth --namespace arc-runners \
   --from-literal=github_app_id=123123 \
   --from-literal=github_app_installation_id=456456 \
   --from-literal=github_app_private_key='-----BEGIN RSA PRIVATE KEY-----
[...]
-----END RSA PRIVATE KEY-----'
```

We create or use an existing “Runner group” set in our Github settings. For the name we will use `arc-runner-set-01` and fill in the details to allow the listener application to authenticate to GitHub Actions service and establish an HTTPS long poll connection.

```shell
INSTALLATION_NAME='arc-runner-set-01'
NAMESPACE='arc-runners-01'
GITHUB_CONFIG_URL='https://github.com/MyOrg'
helm install "${INSTALLATION_NAME}" \
	--namespace "${NAMESPACE}" \
	--create-namespace \
	--set githubConfigUrl="${GITHUB_CONFIG_URL}" \
	--set githubConfigSecret="gha-auth" \
	--set runnerScaleSetName="${INSTALLATION_NAME}" \
	--set runnerGroup="${INSTALLATION_NAME}" \
	oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

* `githubConfigUrl` sets to either an Enterprise, Organization or Repo URL. In this example we link the pod runners to our Organization;
* `githubConfigSecret` is the reference to the secret we create earlier;
* `runnerScaleSetName` will be used as the base name for our kubernetes resources and
* `runnerGroup` will be used to target this replica set in our Actions

We can now see the runner sets are showing in the Organization level runner groups:
<img width="1038" alt="aaaq123123123" src="https://github.com/aaraiu/arc_docs/assets/128513863/d77e2713-bedb-400b-afa4-c735be71aa26">
To use these new runner groups, we set the runs-on key in our Actions YAML configuration to target the pods in our recently deployed scale sets:
```yaml
name: GitHub Actions K8s Demo
run-name: ${{ github.actor }} is testing out GitHub Actions on K8s 🚀🐳
on: [push]
jobs:
  Explore-GitHub-Actions-K8s:
    runs-on: [arc-runner-set-01] # <====
    steps:
    - run: echo "🎉"
```
