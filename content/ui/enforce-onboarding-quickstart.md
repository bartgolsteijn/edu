## Prerequisites

This walkthrough will quickly get Chainguard Enforce installed and running locally — from setting up an example cluster, to drafting a policy and observing how it behaves, to improving the policy, and finally enforcing that policy. If you'd like more information on working with Chainguard Enforce, we encourage you to to check out the [detailed tutorial on Chainguard Academy](https://edu.chainguard.dev/chainguard/chainguard-enforce/chainguard-enforce-kubernetes/chainguard-enforce-user-onboarding/).

Before running Chainguard Enforce locally, you’ll need to ensure you have the following installed:

* **curl** — to retrieve files from the web, follow the relevant [curl download docs](https://curl.se/download.html) for your machine.
* **Docker** — you’ll need [Docker installed](https://docs.docker.com/get-docker/) and running in order to step through this tutorial. 
* **kind** — to create a kind Kubernetes cluster on our laptop, you can download and install kind for your relevant operating system by following the [kind install docs](https://kind.sigs.k8s.io/docs/user/quick-start/#installation).
* **kubectl** — to work with your kind cluster, you can install for your operating system by following the official [Kubernetes kubectl documentation](https://kubernetes.io/docs/tasks/tools/#kubectl).
* For macOS users, you'll need to update to bash version 4 or higher, which is not preinstalled in the machine. Please [follow our guide](https://edu.chainguard.dev/open-source/update-bash-macos/) on how to update your version if you are getting version 3 or below when you run `bash --version`.

## Step 1 — Install chainctl

Our command line interface (CLI) tool, `chainctl`, will help you interact with the account model that Chainguard Enforce provides, enabling you to make queries into the state of your clusters and policies.

Create a new directory called `enforce-demo`.

```sh
mkdir ~/enforce-demo && cd $_
```

To install `chainctl`, we’ll use the `curl` command to pull the application down.

```sh
curl -o chainctl "https://dl.enforce.dev/chainctl_$(uname -s)_$(uname -m)"
```

Move `chainctl` into your `/usr/local/bin` directory.

```sh
sudo mv ./chainctl /usr/local/bin/chainctl
```

Next, elevate the permissions of `chainctl` so that it can execute as needed.

```sh
chmod +x /usr/local/bin/chainctl
```

Finally, alias its path so that you can use `chainctl` on the command line.

```sh
alias chainctl=/usr/local/bin/chainctl
```

You can verify that everything was set up correctly by checking the `chainctl` version.

```sh
chainctl version
```

```
   ____   _   _      _      ___   _   _    ____   _____   _
  / ___| | | | |    / \    |_ _| | \ | |  / ___| |_   _| | |
 | |     | |_| |   / _ \    | |  |  \| | | |       | |   | |
 | |___  |  _  |  / ___ \   | |  | |\  | | |___    | |   | |___
  \____| |_| |_| /_/   \_\ |___| |_| \_|  \____|   |_|   |_____|
chainctl: Chainguard Control

GitVersion:    e01c38b
GitCommit:     e01c38b452ee34e44cf5f8663d7730a2cf69f0c3
GitTreeState:  clean
BuildDate:     2022-09-21T00:10:26Z
GoVersion:     go1.18.6
Compiler:      gc
Platform:      darwin/arm64
```

## Step 2 — Check IAM group

Using `chainctl`, you can check for the ID of your group.

```sh
chainctl iam groups ls -o table
```

You'll receive output in the form of a table with information about your current groups, similar to the following.

```
                     ID                    |       NAME       | DESCRIPTION  
-------------------------------------------+------------------+--------------

  b9adda06841c1d34dfa73d5902ed44b5448b7958 | enforce-demo-group |              
```

> **Note**: If you don't receive output like the above at all, you can create a new group by running `chainctl iam groups create --no-parent` to create a new group. After group creation, you can run `chainctl iam groups ls -o table` again.

Next, create a variable that stores the ID in the left column for later steps in this tutorial. In the following command, replace `$GROUP_ID` with the relevant ID.

```sh
export GROUP=$GROUP_ID
```

In the UI, you can also check for groups to which you belong from the filter modal, which you can open by clicking on the filter icon in the top-level navigation menu.

![Group dropdown](/images/group-dropdown.png)

You can check here to see the groups to which you belong and filter resources based on group ownership.

## Step 3 — Prepare Kubernetes cluster

In order to put Chainguard Enforce into action within a cluster, we'll now create a Kubernetes cluster using kind. We will name our cluster `enforce-demo` by passing that to the `--name` flag, but you may want to use another name.

```sh
kind create cluster --name enforce-demo
```

Install the Chainguard Enforce agent in your cluster:

```sh
chainctl cluster install --group=$GROUP --private --context kind-enforce-demo
```

If you click on the [**Clusters** tab](https://console.enforce.dev/clusters) in the main navigation menu, you should now see your cluster in the cluster table.

![Cluster list](/images/cluster-list.png)

From here, you can explore a detailed view of the cluster, including any policies that apply to it.

## Step 4 — Create a security policy

You can create a policy directly from the UI by navigating to the [**Policies** tab](https://console.enforce.dev/policies). In the policy table menu, there will be a **Create policy** button. Clicking this button will open a dropdown menu, which will allow you to create a policy from scratch or use a predefined template.

For now, we can create a policy using the [**Custom** option from the dropdown](https://console.enforce.dev/policies/create/custom).

![Create policy](/images/create-policy.png)

On the policy create page, ensure that the correct group is displayed in the group field: `enforce-demo`. Then paste the following code into the code editor:

```
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: sample-policy
spec:
  images:
  - glob: "ghcr.io/chainguard-dev/*/*"
  - glob: "ghcr.io/chainguard-dev/*"
  - glob: "index.docker.io/*"
  - glob: "index.docker.io/*/*"
  authorities:
  - keyless:
      url: https://fulcio.sigstore.dev
```

This policy creates a cluster image policy with the Sigstore beta API, and with Fulcio as a keyless authority. Here, we are requiring that all images from container registries be signed.

After you click the **Publish** button at the bottom of the modal, your new policy will be active. The next time you land on the policy list page, you will see the policy listed, as well as any violations it has and its group hierarchy.

You can also list your policies with `chainctl`.

```sh
chainctl policies ls
```

A few policies will be in place in addition to the policy you just created.

## Step 5 — Inspect compliance of containers

The first place we can look for information about container compliance is the clusters main page, which you can find by clicking on the [**Clusters** tab](https://console.enforce.dev/clusters) in the main navigation menu.

With our new policy, `sample-policy`, in place, information about policy compliance should be visible in the **Policy** column, next to the cluster name.

![Cluster compliance](/images/cluster-compliance.png)

You can also find more information about policy compliance by clicking on either of the cards in the cluster list page. The links on these cards will take you to views that provide more information on policies that have failed, and the exact images that are failing policies.

Additionally, the buttons on top of the cluster table will allow you to filter your clusters by compliance.

You can also check that the **sample-policy** was distributed to the cluster by using `kubectl`.

```sh
kubectl get clusterimagepolicies
```

You’ll get feedback that the **sample-policy** was distributed and how long ago.

```
NAME                AGE
sample-policy     68s
```

## Step 6 — Test new record compliance

So far, the information we have been seeing about our cluster concerns the containers and images running in the cluster's control plane.

Now, let’s deploy a new image, starting with a generic NGINX image.

```sh
kubectl create deployment nginx --image=nginx
```

Give this a few seconds to populate and then check what’s running again by navigating to the cluster's detail page. You can do this by clicking on the cluster's name from the cluster list table.

In the **Policy violations** table, the image will be listed.

![Image with violations](/images/problem-nginx.png)

Clicking on the **Show diagnostic** button in the table will provide more information about the violation.

Next, let’s pull in an image that has an SBOM and signature. This is an NGINX image from Chainguard.

```sh
kubectl create deployment good-nginx --image=ghcr.io/chainguard-dev/nginx-image-demo
```

This image won't be listed in the violations table, but you can navigate to the **Images** table to retrieve it.

![Image without violations](/images/good-nginx.png)
 
This image passes the policy because it has both an SBOM and a signature.

## Step 7 – Enforce policy

At this point, let’s enforce this policy requirement. We can use `kubectl` and the `namespace` label selectors to do this.

```sh
kubectl label ns default policy.sigstore.dev/include=true --overwrite
```

We can check that our policy is enforced by trying to run an unsigned image. We’ll use an unsigned Ubuntu image as an example.

```sh
kubectl run not-signed --image=ubuntu
```

You’ll receive output that this attempt at running an unsigned image has been rejected.

```
Error from server (BadRequest): admission webhook "enforcer.chainguard.dev" denied the request: validation failed: failed policy: sample-policy
```

Congratulations, you have completed onboarding to Chainguard Enforce!

If you would like, you can now clean up your work by uninstalling `chainctl` and then deleting the cluster.

```sh
chainctl cluster uninstall
```

```sh
kind delete cluster --name enforce-demo
```

To learn more about Chainguard Enforce, please review our documentation and other resources on Chainguard Academy.
