# amqstreams-argocd

This demo deploys a very small AMQ Streams cluster (one single ZK and one single Kafka with low resources comsumption) using ArgoCD.

## Install operators

There are two available YAML files in the repository for installing both operators:

* AMQ Streams operator: ```install-amqs-operator.yaml```
* OpenShift GitOps operator: ```install-gitops-operator.yaml```

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

Default creater username is ```admin```.

```bash
$ ARGO_USER=admin

$ ARGO_ROUTE=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}')

$ ARGO_PASS=$(oc get secret/openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)

$ argocd login $ARGO_ROUTE --insecure --username $ARGO_USER --password $ARGO_PASS
```

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

Then, apply the manifest:

```bash
$ oc apply -f argocd-app-manifests/amq-streams/test/manifest.yaml
```

Check deployed Manifest:

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

4. Check the application status

```bash
$ argocd app get amq-streams-test
```

### Deploy using CLI

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

You can see detailed parameters and example for this action, usin the command help:

```bash
$ argocd app create --help
```

The following CLI command will deploy the same application configuration that is defined above, in the Application Manifest.

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

## Deploy AMQ Streams clusters (two environments)

```bash
$ argocd proj create amq-streams-project

$ argocd app create \
  --name amq-streams-dev \
  --dest-namespace amq-streams-dev \
  --dest-server https://kubernetes.default.svc \
  --project amq-streams-project \
  --repo https://github.com/ryanezil/amqstreams-argocd.git \
  --path amq-streams/overlays/dev \
  --revision main \
  --sync-policy automated \
  --auto-prune \
  --sync-option CreateNamespace=true

$ argocd app create \
  --name amq-streams-test \
  --dest-namespace amq-streams-test \
  --dest-server https://kubernetes.default.svc \
  --project amq-streams-project \
  --repo https://github.com/ryanezil/amqstreams-argocd.git \
  --path amq-streams/overlays/test \
  --revision main \
  --sync-policy automated \
  --auto-prune \
  --sync-option CreateNamespace=true
```

## Managing Users and Topics

How to manage users and topics, for every different application working with AMQ Streams.

Now the parameter ```--sync-option CreateNamespace=true``` is not used: target namespace must be created when the cluster is deployed.

### Test environment

* **Application 1 resources**

```bash
$ argocd app create \
  --name amq-streams-test-application1 \
  --dest-namespace amq-streams-test \
  --dest-server https://kubernetes.default.svc \
  --project default \
  --repo https://github.com/ryanezil/amqstreams-argocd.git \
  --path amq-streams-applications/application1 \
  --directory-recurse \
  --revision main \
  --sync-policy automated \
  --auto-prune
```

* **Application 2 resources**

```bash
$ argocd app create \
  --name amq-streams-test-application2 \
  --dest-namespace amq-streams-test \
  --dest-server https://kubernetes.default.svc \
  --project default \
  --repo https://github.com/ryanezil/amqstreams-argocd.git \
  --path amq-streams-applications/application2 \
  --directory-recurse \
  --revision main \
  --sync-policy automated \
  --auto-prune
```

