# amqstreams-argocd

This project is an example using Argo CD for deploying a very small AMQ Streams cluster on OpenShift.

Also it shows a basic approach for managing two clusters and different application resources (users & topics).

Since v2.0, Argo CD includes health checks for Strimzi CRDs:
* Kafka
* KafkaTopic
* KafkaUser
* KafkaConnect

## Install operators

There are two available YAML files in the repository for installing both operators:

* AMQ Streams operator: ```install-amqs-operator.yaml```
* OpenShift GitOps operator: ```install-gitops-operator.yaml```

```bash
$ oc create -f install-gitops-operator.yaml

subscription.operators.coreos.com/openshift-gitops-operator created
```

After RH OpenShift GitOps operator is installed, you can find a new project recently created:

```bash
$ oc describe project openshift-gitops
Name:               openshift-gitops
Created:            11 minutes ago
Labels:             olm.operatorgroup.uid/2f9ef96e-4bc0-4311-8881-e86c51f3bc99=
                    openshift.io/cluster-monitoring=true
Annotations:        openshift.io/sa.scc.mcs=s0:c25,c5
                    openshift.io/sa.scc.supplemental-groups=1000610000/10000
                    openshift.io/sa.scc.uid-range=1000610000/10000
Display Name:       <none>
Description:        <none>
Status:             Active
Node Selector:      <none>
Quota:              <none>
Resource limits:    <none>
```

With the following PODs running:

```bash
$ oc get pods -n openshift-gitops

NAME                                                        READY   STATUS    RESTARTS   AGE
cluster-5b574cff45-c58q7                                    1/1     Running   0          17m
kam-7f65f49f56-n9xt8                                        1/1     Running   0          17m
openshift-gitops-application-controller-0                   1/1     Running   0          17m
openshift-gitops-applicationset-controller-769bc45f-tgt8x   1/1     Running   0          17m
openshift-gitops-redis-7765dd9fc9-9lqt6                     1/1     Running   0          17m
openshift-gitops-repo-server-7c46884cf6-vn5rm               1/1     Running   0          17m
openshift-gitops-server-7975f7b985-rf4p2                    1/1     Running   0          17m
```

### Configure ArgoCD ServiceAccount

Allow the serviceAccount for ArgoCD the ability to manage the cluster:

```bash
$ oc adm policy add-cluster-role-to-user cluster-admin -z openshift-gitops-argocd-application-controller -n openshift-gitops
```

You will see an errors like this when ArgoCD tries to create AMQ Streams resources, if you don't exec this step:

```bash
kafkas.kafka.strimzi.io is forbidden: User "system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller" cannot create resource "kafkas" in API group "kafka.strimzi.io" in the namespace "amq-streams-test"
```

## Configure ArgoCD

### Install CLI

The following steps are extracted from here: https://argoproj.github.io/argo-cd/cli_installation/

0. Check latest version available at https://github.com/argoproj/argo-cd/releases/latest

1. You can view the latest version of Argo CD at the link above or run the following command to grab the version:
    ```bash
    VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
    ```
2. Replace VERSION in the command below with the version of Argo CD you would like to download:
    ```bash
    curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
    ```
3. Make the argocd CLI executable:
    ```bash
    chmod +x /usr/local/bin/argocd
    ```
4. To enable bash completion, run the following command:
    ```bash
    source <(argocd completion bash)
    ```

You should now be able to run argocd commands

### Login to ArgoCD

Default username is ```admin```.

```bash
$ ARGO_USER=admin

$ ARGO_ROUTE=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}')

$ ARGO_PASS=$(oc get secret/openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)

$ argocd login $ARGO_ROUTE --insecure --username $ARGO_USER --password $ARGO_PASS
```

#### Configure SSO

> Read the this [OpenShift blog](https://www.openshift.com/blog/sso-integration-for-the-openshift-gitops-operator) showing a step-by-step guide on using Red Hat Single Sign-On(RHSSO) to log in to an Argo CD application 

### Check Managed clusters

The following output shows the clusters that Argo CD manages.

```bash
$ argocd cluster list
SERVER                          NAME        VERSION  STATUS   MESSAGE
https://kubernetes.default.svc  in-cluster           Unknown  Cluster has no application and not being monitored.
```
The NAME ```in-cluster``` means that Argo CD is managing the cluster it's installed on.

### Access ArgoCD WebConsole

Retrieve exposed URL:

```bash
$ oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}'
```
Get ```admin``` user password:

```bash
$ oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
# admin.password
DXsdeiAbgrIzh3wtFcH1Sfmy2OV40MkR
```

## Deploy an application with ArgoCD

### Deploy using an Application Manifest

1. Create the application manifest file

    You need to create a manifest file containing all the application definition:

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: amq-streams-test
      namespace: openshift-gitops
    spec:
      destination:
        namespace: amq-streams-test
        server: https://kubernetes.default.svc
      project: default
      source:
        path: amq-streams/overlays/test
        repoURL: https://github.com/ryanezil/amqstreams-argocd
        targetRevision: main
      syncPolicy:
        automated:
          prune: true
          selfHeal: false
        syncOptions:
        - CreateNamespace=true
    ```

2. Apply the manifest:

    ```bash
    $ oc apply -f argocd-app-manifests/amq-streams/test/manifest.yaml
    ```

3. Check deployed Manifest:

    ```bash
    $ argocd app manifests amq-streams-test
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      labels:
        app.kubernetes.io/instance: amq-streams-test
      name: amq-streams-test

    ---
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    metadata:
      labels:
        app.kubernetes.io/instance: amq-streams-test
      name: single-node-cluster
      namespace: amq-streams-test
    spec:
      entityOperator:
        tlsSidecar:
          resources:
            limits:
              :
              :
              :
              :
    ```

    Now you can check the application status

    ```bash
    $ argocd app get amq-streams-test
    ```

4. Clean and remove the entire application

    ```bash
    $ oc delete application amq-streams-test -n openshift-gitops

    application.argoproj.io "amq-streams-test" deleted
    ```

### Deploy using CLI commands

The following steps show how to deploy an application using the command line.

1. Add remote application repository to ArgoCD

    ```bash
    $ argocd repo add https://github.com/ryanezil/amqstreams-argocd.git

    Repository 'https://github.com/ryanezil/amqstreams-argocd.git' added
    ```

2. Check the repo has been added:

    ```bash
    $ argocd repo list

    TYPE  NAME  REPO                                               INSECURE  OCI    LFS    CREDS  STATUS      MESSAGE
    git         https://github.com/ryanezil/amqstreams-argocd.git  false     false  false  false  Successful
    ```

3. Create the application

    You can see detailed parameters and examples for this action, usin the command help:

    ```bash
    $ argocd app create --help
    ```

    The following CLI command will deploy the same application configuration that is defined above in the Application Manifest example.

    ```bash
    $ argocd app create \
      --name amq-streams-test \
      --dest-namespace amq-streams-test \
      --dest-server https://kubernetes.default.svc \
      --project default \
      --repo https://github.com/ryanezil/amqstreams-argocd.git \
      --path amq-streams/overlays/test \
      --revision main \
      --sync-policy automated \
      --auto-prune \
      --sync-option CreateNamespace=true


    application 'amq-streams-test' created
    ```

4. Check the application status

    ```bash
    $ argocd app get amq-streams-test

    Name:               amq-streams-test
    Project:            default
    Server:             https://kubernetes.default.svc
    Namespace:          amq-streams-test
    URL:                https://openshift-gitops-server-openshift-gitops.apps-crc.testing/applications/amq-streams-test
    Repo:               https://github.com/ryanezil/amqstreams-argocd.git
    Target:             main
    Path:               amq-streams/overlays/test
    SyncWindow:         Sync Allowed
    Sync Policy:        Automated (Prune)
    Sync Status:        Synced to main (35d5af0)
    Health Status:      Healthy

    GROUP             KIND       NAMESPACE         NAME                 STATUS   HEALTH   HOOK  MESSAGE
                      Namespace  amq-streams-test  amq-streams-test     Running  Synced         namespace/amq-streams-test created
    kafka.strimzi.io  Kafka      amq-streams-test  single-node-cluster  Synced   Healthy        kafka.kafka.strimzi.io/single-node-cluster created
                      Namespace                    amq-streams-test     Synced
    ```

5. Remove the application using the CLI

    ```bash
    $ argocd app delete amq-streams-test

    Are you sure you want to delete 'amq-streams-test' and all its resources? [y/n]
    ```

### Deploy using Web Console

Out of Scope

## Deploy AMQ Streams Environments

Projects provide a logical grouping of applications, which is useful when Argo CD is used by multiple teams. Projects provide the following features:


* restrict what may be deployed (trusted Git source repositories)
* restrict where apps may be deployed to (destination clusters and namespaces)
* restrict what kinds of objects may or may not be deployed (e.g. RBAC, CRDs, DaemonSets, NetworkPolicy etc...)
* defining project roles to provide application RBAC (bound to OIDC groups and/or JWT tokens)

> **Source:** Argo CD Projects [documentation](https://argoproj.github.io/argo-cd/user-guide/projects/).

You can create a project using a ```AppProject``` resource instead of using cli commands. See the link: https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#app-of-apps

1. Create two ArgoCD projects: DEV and TEST.

    The following commands create two new projects with the same configuration used for the 'default' Argo CD project.

    **DEV environment**

    ```bash
    $ argocd proj create amq-streams-dev --description 'AMQ Streams DEV'

    $ argocd proj add-source amq-streams-dev '*'
    $ argocd proj add-destination amq-streams-dev '*' '*'
    $ argocd proj allow-cluster-resource amq-streams-dev '*' '*'
    ```
    **TEST environment**

    ```bash
    $ argocd proj create amq-streams-test --description 'AMQ Streams TEST'

    $ argocd proj add-source amq-streams-test '*'
    $ argocd proj add-destination amq-streams-test '*' '*'
    $ argocd proj allow-cluster-resource amq-streams-test '*' '*'
    ```

2. Check project information

    ```bash
    $ argocd proj get amq-streams-dev

    Name:                        amq-streams-dev
    Description:                 AMQ Streams DEV
    Destinations:                *,*
    Repositories:                *
    Allowed Cluster Resources:   */*
    Denied Namespaced Resources: <none>
    Signature keys:              <none>
    Orphaned Resources:          disabled
    ```

3. Check OpenShift project resource

    ```bash
    $ oc describe appproject amq-streams-dev -n openshift-gitops
    ```

    Output:

    ```bash
    Name:         amq-streams-dev
    Namespace:    openshift-gitops
    Labels:       <none>
    Annotations:  <none>
    API Version:  argoproj.io/v1alpha1
    Kind:         AppProject
    Metadata:
      Creation Timestamp:  2021-06-28T15:26:01Z
      Generation:          4
      Managed Fields:
        API Version:  argoproj.io/v1alpha1
        Fields Type:  FieldsV1
        fieldsV1:
          f:spec:
            .:
            f:clusterResourceWhitelist:
            f:description:
            f:destinations:
            f:sourceRepos:
          f:status:
        Manager:         argocd-server
        Operation:       Update
        Time:            2021-06-28T15:26:11Z
      Resource Version:  145067
      Self Link:         /apis/argoproj.io/v1alpha1/namespaces/openshift-gitops/appprojects/amq-streams-dev
      UID:               2d61c1da-4ba5-4b30-ad54-d53dce0607e0
    Spec:
      Cluster Resource Whitelist:
        Group:      *
        Kind:       *
      Description:  AMQ Streams DEV
      Destinations:
        Namespace:  *
        Server:     *
      Source Repos:
        *
    Status:
    Events:
      Type    Reason           Age   From           Message
      ----    ------           ----  ----           -------
      Normal  ResourceCreated  60m   argocd-server  admin created project
      Normal  ResourceUpdated  60m   argocd-server  admin updated project
      Normal  ResourceUpdated  60m   argocd-server  admin updated project
      Normal  ResourceUpdated  60m   argocd-server  admin updated project
    ```

4. Deploy AMQ Streams clusters

    TEST environment configuration is changing some of the ```base``` configuration values defined:
    * Number of replicas for Zookeeper and Kafka is now 2 (base value is 1)
    * Default number of partitions is 5 (base value is 3)

    ```bash
    # AMQ Streams DEV cluster
    $ argocd app create \
      --name amq-streams-dev \
      --dest-namespace amq-streams-dev \
      --dest-server https://kubernetes.default.svc \
      --project amq-streams-dev \
      --repo https://github.com/ryanezil/amqstreams-argocd.git \
      --path amq-streams/overlays/dev \
      --revision main \
      --sync-policy automated \
      --auto-prune \
      --sync-option CreateNamespace=true

    # AMQ Streams TEST cluster
    $ argocd app create \
      --name amq-streams-test \
      --dest-namespace amq-streams-test \
      --dest-server https://kubernetes.default.svc \
      --project amq-streams-test \
      --repo https://github.com/ryanezil/amqstreams-argocd.git \
      --path amq-streams/overlays/test \
      --revision main \
      --sync-policy automated \
      --auto-prune \
      --sync-option CreateNamespace=true
    ```

## Managing Users and Topics

This section shows how to manage users and topics used by every different application producing and consuming messages.

Now the parameter ```--sync-option CreateNamespace=true``` is not used: target namespace must be created when the cluster is deployed.

### Test environment

Only ```Application 1``` is available in TEST.

> The ```revision```parameter points to the Git branch ```main``` (you could use a Tag, instead).

* **Application 1 resources**

```bash
$ argocd app create \
  --name amq-streams-application1-test \
  --dest-namespace amq-streams-test \
  --dest-server https://kubernetes.default.svc \
  --project amq-streams-test \
  --repo https://github.com/ryanezil/amqstreams-argocd.git \
  --path amq-streams-applications/application1 \
  --directory-recurse \
  --revision main \
  --sync-policy automated \
  --auto-prune
```

### Develop environment

Both ```Application 1``` and ```Application 2``` are available in DEVELOP. Additionally, Aplication1 has one more user created in this environment, that is not created in TEST.

> The ```revision```parameter points to the Git branch ```develop```.

* **Application 1 resources**

```bash
$ argocd app create \
  --name amq-streams-application1-dev \
  --dest-namespace amq-streams-dev \
  --dest-server https://kubernetes.default.svc \
  --project amq-streams-dev \
  --repo https://github.com/ryanezil/amqstreams-argocd.git \
  --path amq-streams-applications/application1 \
  --directory-recurse \
  --revision develop \
  --sync-policy automated \
  --auto-prune
```

* **Application 2 resources**

```bash
$ argocd app create \
  --name amq-streams-application2-dev \
  --dest-namespace amq-streams-dev \
  --dest-server https://kubernetes.default.svc \
  --project amq-streams-dev \
  --repo https://github.com/ryanezil/amqstreams-argocd.git \
  --path amq-streams-applications/application2 \
  --directory-recurse \
  --revision develop \
  --sync-policy automated \
  --auto-prune
```

![Dev environment](./images/argo-cd.png "DEV environment")
