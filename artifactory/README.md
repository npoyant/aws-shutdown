# Artifactory
TSSC Artifactory Deployment Documentation

## Deployment


### Install OpenShift Client 
If you do not have the oc client installed, refer to the following documentation

https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html#installing-the-cli

### Login to OpenShift
Login to OCP w/ an account that has cluster admin level permissions

```
oc login
```


### Create Artifactory OpenShift Project
Create an OpenShift Project for Artifactory (Can be configured to any name)

```
oc adm new-project artifactory
```

```
oc project artifactory
```

### Install Helm Client
If you do not have the helm client installed, refer to the following documentation:

https://docs.openshift.com/container-platform/latest/cli_reference/helm_cli/getting-started-with-helm-on-openshift-container-platform.html#installing-helm_getting-started-with-helm-on-openshift

### Configure Helm Repository
A helm repository for JFROG will need to be configured to access the helm charts

```
helm repo add jfrog https://charts.jfrog.io/
```
```
helm repo update
```

### Config Custom Security Context Constraint (SCC)
The current helm charts provided by JFROG will fail by default during an OpenShift deployment. The failure is due to having hard coded UID/GID's that are not compliant during SCC checks. Create a new `anyuid` scc for your artifactory `serviceaccount` to allow these pods to run under their specified UID while minimizing any risk of using the system anyuid SCC:

```
oc process -f https://raw.githubusercontent.com/redhat-cop/openshift-templates/master/scc/project-run-anyuid-template.yml NAMESPACE=artifactory NAME=artifactory-oss-artifactory | oc apply -f -
```

### Deploy Artifactory
Utilizing helm carts, deploy artifactory to OpenShift
```
helm upgrade --install artifactory-oss --set artifactory.nginx.enabled=false --set artifactory.postgresql.postgresqlPassword=artifactory --namespace artifactory jfrog/artifactory-oss
```
**Note:** Multiple attempts were made with various cli flags and a bootstrap values file in order to set the initial password but it would never set. A git issue will be submitted against the project to track the issue.

### Update Statefulset to utilize a Service Account that can access the Custom SCC
The `statefulset` needs to be updated to utilize the authorized service account in order for the pods to start properly

```
oc patch statefulset.apps/artifactory-oss-postgresql --patch '{"spec":{"template":{"spec":{"serviceAccountName": "artifactory-oss-artifactory"}}}}'
```

### Create a route in order to access the Artifactory UI

```
oc expose svc artifactory-oss-artifactory --name="artifactory" --hostname="artifactory.apps.tssc.rht-set.com"
```

### Change Default Password
**Note:** When all pods enter a ready state, login to the artifactory page w/ the default `user: admin pw: password`. You are prompted to change the pw at this time: Enter New PW
