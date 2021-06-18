# amqstreams-argocd

## Install operators

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

You will see an error like this in ArgoCD if you don't exec this step:

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

## Deploy using Application Manifest

```bash
$ oc apply -f argocd-app-manifests/amq-streams/test/manifest.yaml
```

