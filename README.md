# kafka-devops

Simulated production environment running a streaming application targeting Apache Kafka on Confluent Cloud.
Applications and resources are managed by GitOps with declarative infrastructure, Kubernetes and the [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

For the full documentation on the the project see https://docs.confluent.io (TODO: real confluent docs link here).

For basic usage instructions on how to use this project, read on.

# Tool prerequisites

The included `Makefile` contains a target to install these on MacOS or they can be installed manually. Details for automated installation in the Usage section.

## ccloud
Required to interact with the Confluent Cloud components managed by this project

https://docs.confluent.io/current/cloud/cli/install.html

## kubectl
Required to interact with your Kubernetes cluster, create secrets, etc...

https://kubernetes.io/docs/tasks/tools/install-kubectl/

## k3d
If you'd like to run the project on a local Docker based Kubernetes cluster

https://github.com/rancher/k3d#get

## Kustomize
Environments (dev, stg, prd, etc...) are created by using Kustomize style overlays

https://github.com/kubernetes-sigs/kustomize

## Helm 3
Used to install Flux into the cluster

https://helm.sh/docs/intro/install/

## Bitnami Sealed Secrets Kubeseal
The repository uses sealed secrets and the Bitnami controller for managing secret values. `kubeseal` is used in scripting to seal the secrets locally, which are then committed to the repository.

https://github.com/bitnami-labs/sealed-secrets

## jq
`jq` is used for processing JSON

https://stedolan.github.io/jq/

## yq
`yq` is used for processing YAML

https://github.com/mikefarah/yq

# Confluent Cloud

This demo utilizes Confluent Cloud for Kafka, Schema Registry, ksqlDB services. In order to run this demo you will need a Confluent Cloud account, the `ccloud` CLI, as well as an environment and Kafka cluster setup. 

In the future, all steps (besides account creation), will be automated and managed by operations in this repository. Until then, there are some setup steps to make the application function properly.

_*TODO*_: Link to Docs for setting up ccloud and environment properly until automated...

# Usage 

If you'd like to run a version of this repository in your own cluster, follow the below usage steps. We also plan to build and manage a deployment of this repository ourselves in order to allow public PRs to affect a running instance for demonstration purposes.

1.  Fork this repository

2.  Update the following variables in `scripts\flux-init.sh`

  * `ENVIRONMENT=dev` You'll complete this process for each environment you want.
  * `REPO_URL=git@github.com:your-fork/kafka-devops` Update to match your git remote URL
  * `REPO_GIT_USER=your-user` Update to your git username
  * `REPO_GIT_EMAIL=your-user@example.com` Update to your git email

3.  Install all dependencies

    On macOS a provided `make` target uses a combination of manual installations by downloading and install binaries to `/usr/local/bin` and Homebrew. You will be prompted for your adminstrative passwod to install files to `/usr/local/bin`.  You can skip this step if you'd like to install the dependencies manually.

    ```
    make install-deps 
    ```

4. Create a local test cluster on Docker using k3d

    If you want to run a demo of this project on a local K8s cluster, this `make` target can boostrap a `k3d` cluster for you.

    ```
    make cluster
    ```

    Verify the cluster is ready:

    ```
    kubectl get nodes

    NAME                        STATUS   ROLES    AGE   VERSION
    k3d-kafka-gitops-server-0   Ready    master   24s   v1.18.4+k3s1
    k3d-kafka-gitops-server-1   Ready    master   15s   v1.18.4+k3s1
    k3d-kafka-gitops-server-2   Ready    master   12s   v1.18.4+k3s1
    k3d-kafka-gitops-server-3   Ready    master   10s   v1.18.4+k3s1 
    ```

5. Install Bitnami Sealed Secret Controller into the cluster

    Once you have a cluster configured for your local `kubectl` command, the following will install the secret controller

    ```
    make install-bitnami-secret-controller
    ```

    Wait for the controller to be ready. This will transition from `null` to `1` (available replica):

    ```
    kubectl get -n kube-system deployment/sealed-secrets-controller -o json | jq '.status.availableReplicas'
    ```

6. Retrieve the secret controller's public key for this environment

    The public key is used to seal the secrets which are then committed to the Git repository.  Only the secret controller (which generated or is configured with this public key) can decrypt the sealed secrets inside the cluster.
   
    If your cluster has public nodes (which is true for the local dev cluster setup in these instructions), you can obtain and save the public key using:

    ```
    make get-public-key-dev
    ```
  
    If you are using a private cluster, you will need to copy the secret controller's key from the secret controller's log file into the key file stored locally.  However you obtain the public key, it can be stored in `secrets/keys/<environment>.crt`, _but not checked into the repository_. The remaining scripts look in this location in order to seal secrets. If you have administrative login to the cluster with `kubectl`, you may be able to get the logs by executing the following substituting your controllers full pod name (`kubectl get pods -n kube-system`):
  
    ```
    kubectl logs sealed-secrets-controller-6bf8c44ff9-x6skc -n kube-system
    ```
  
    See the Bitnami docs for long term management of secrets and more details on private clusters (https://github.com/bitnami-labs/sealed-secrets/blob/master/docs/GKE.md#private-gke-clusters).

7. Create and deploy the sealed secrets

    There are two external secrets required to utilize this project.  The following helps you create two secret files seal them for use inside the cluster. The namespace, secret name, and generic secret file name are related in the following commands, do not change them without understanding the seal script, executed next.

    * `ccloud` CLI login credentials are used to manage the Confluent Cloud resources controlled using the [ccloud operator code](images/ccloud-operator). An example of the layout of the secrets file required can be found in the file [secrets/example-ccloud-secrets.props](secrets/example-ccloud-secrets.props).  Create a local secrets files for your `ccloud` credentials, _ensuring you do not commit them to any repository_. Execte a `kubectl create secret` command as below passing the path to your `ccloud` credentials file into the `--from-env-file` argument. 

      ```
      kubectl create secret generic cc.ccloud-secrets --namespace=default --from-env-file=<path-to-your-file> --dry-run=client -o yaml > secrets/local-toseal/dev/default-ccloud-secrets.yaml
      ```


    * The microservices demo code utilizes a MySQL database to demonstrate Kafka Connect and Change Data Capture. Credentials for the database are required to be provided.  An example of the layout of this file can be found in the sample [secrets/example-connect-operator-secrets.props](secrets/example-connect-operator-secrets.props). There isn't a need to create a seperate file for the database credentials file as that service is ran entirely inside the demonstrations Kubernetes cluster and is not publically accessible.

      ```
      kubectl create secret generic connect-operator-secrets --namespace=default --from-env-file=./secrets/example-connect-operator-secrets.props --dry-run=client -o yaml > secrets/local-toseal/dev/default-connect-operator-secrets.yaml 
      ```

    These commands have created generic secret Kubernetes secret manifests from your secrets files and put them into a staging area (`secrets/local-toseal`).  

  * Seal the secrets, for the `dev` environment, with the following helper command which uses the `scripts/seal-secrets.sh` script. This command will place the sealed secret in `secrets/sealed/dev`, and these are the files which are safe to commit to the repository.

    ```
    make seal-secrets-dev
    ```

  * Commit the sealed secret to the repository so that Flux can sync it to the K8s cluster:

    ```
    git add secrets/sealed/dev/.
    git commit -m 'New secrets!'
    git push origin master # (or to the appropriate branch if you are doing GitOps by PR already!)
    ```

8. Install Flux, the GitOps operator, into the cluster

    ```
    make install-flux
    ```

  * The script will install Flux into the cluster and then wait for you to add the shown key to your repository in the Settings->Deploy Keys section. Write access is required for Flux to manage Tags to control the syncronized state.  See the Flux documentation for more details.

  * You will see the following if the syncronization between Flux and the repository is setup properly:

      ```
      >>> Github deploy key is ready
      >>> Cluster bootstrap done!
      ```

9. Verify secrets are available

    ```
    kubectl get sealedsecrets.bitnami.com
    kubectl get secrets
    ```

    Combining `kubectl`, `jq`, and `base64`, you can decode the secret file to ensure it has been properly set. This should match the original properties file you created. It's controlled by the GitOps process in the same way the other K8s manifests are, however it's not exposed in the code repository.

    ```
    kubectl get secrets/kafka-secrets -o json | jq -r '.data."kafka.properties"' | base64 -d
    ```

10. Verify the system is deployed

    ```
    kubectl get all
    ```

## Info

* FluxCD is configured to sync with the repository once per minute, you can force a syncronization with the command: `make sync`
* You can open a terminal on the cluster with some dev tools available with: `make prompt`

## Credits / Links
* Significant portions of the repository are based on the work of Steven Wade @ https://github.com/swade1987
* The script based Operator patterns in this repository are based on the shell-operator project @ https://github.com/flant/shell-operator
* [FluxCD](https://github.com/fluxcd/flux) is used for GitOps based CD
