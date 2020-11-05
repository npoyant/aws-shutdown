# cert-manger

This is some rough run-book style notes on how cert-manager was configured on the tssc-infra OCP 4 cluster

## install
Tried installing the operator, it wanted to come from the marketplace, couldn't figure out how to get that configured, gave up and used the manual install.

https://cert-manager.io/docs/installation/openshift/#installing-with-regular-manifests
```bash
oc apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml
```

## setup aws

so that wildcard certificates can be granted aws needs to be configured.

This was put togeather with a combination of the following resources:
* https://medium.com/faun/wildcard-k8s-4998173b16c8
* https://cert-manager.io/docs/configuration/acme/dns01/route53/
* https://rcarrata.com/openshift/ocp4\_renew\_automatically\_certificates/

### Set up aws iam account
```bash
AWS_CERT_MANAGER_POLICY_NAME=ocp-cert-manager
AWS_CERT_MANAGER_GROUP_NAME=ocp-cert-manager
AWS_CERT_MANAGER_USERNAME=ocp-tssc-infra-cert-manager

aws iam create-policy --policy-name ${AWS_CERT_MANAGER_POLICY_NAME} --policy-document file://aws-policy-letsencrypt-wildcard.json --profile tssc-infra
LE_POLICY_ARN=$(aws iam list-policies --output json --query 'Policies[*].[PolicyName,Arn]' --output text --profile tssc-infra | grep ${AWS_CERT_MANAGER_POLICY_NAME} | awk '{print $2}')
aws iam create-group --group-name ${AWS_CERT_MANAGER_GROUP_NAME} --profile tssc-infra
aws iam attach-group-policy --policy-arn ${LE_POLICY_ARN} --group-name ${AWS_CERT_MANAGER_GROUP_NAME} --profile tssc-infra
aws iam create-user --user-name ${AWS_CERT_MANAGER_USERNAME} --profile tssc-infra
aws iam add-user-to-group --user-name ${AWS_CERT_MANAGER_USERNAME} --group-name ${AWS_CERT_MANAGER_GROUP_NAME} --profile tssc-infra
aws iam create-access-key --user-name ${AWS_CERT_MANAGER_USERNAME} --profile tssc-infra
```

### setup secret with private key

the private key was put into ocp-infra-cert-manager-aws-secret-access-key.txt encrypted with the PGP keys from [tssc-infra/public-pgp-keys](https://github.com/rhtconsulting/tssc-infra/tree/main/public-pgp-keys).

```bash
sops -d ocp-infra-cert-manager-aws-secret-access-key.txt > ocp-infra-cert-manager-aws-secret-access-key.txt.dec
oc create secret generic aws-route53-creds-${AWS_CERT_MANAGER_USERNAME} --from-file=ocp-infra-cert-manager-aws-secret-access-key.txt.dec -n cert-manager
rm -f ocp-infra-cert-manager-aws-secret-access-key.txt.dec
```

## setup cert-manager
Since openshift's default nameresolver will say that openshift is the authoritive DNS for *.apps.my-cluster you have to configure cert-manager to use a public DNS for resolution to verify that the challange DNS entry is created in route53. For more infor see:
* https://github.com/jetstack/cert-manager/issues/1515
* https://cert-manager.io/docs/configuration/acme/dns01/#setting-nameservers-for-dns01-self-check

Since not using operator just hand jamed --dns01-recursive-nameservers="8.8.8.8:53,1.1.1.1:53" into the cert-manger dc. In theory the paramters could have been set during install time to, but i didn't back track to figure that out.
```bash
oc edit dc/cert-manager -n cert-manager
```

## setup ClusterIssuer
Hand jamed in the hostedzone and aws iam account info into the CLusterIssuer resource.

```bash
oc apply -f ClusterIssuer_letsencrypt-prod.yml
```

## create certificates
So far two certifictes are being managed by cert-manager

* Certificate_quay-quay-enterprise-apps-tssc-rht-set-com.yml
* Certificate_wildcard_apps-tssc-rht-set-com.yml

```bash
oc apply -f Certificate_wildcard_apps-tssc-rht-set-com.yml -n openshift-ingress
oc apply -f Certificate_quay-quay-enterprise-apps-tssc-rht-set-com.yml -n quay-enterprise
```

## un-install

### cert-manager
```bash
oc delete project cert-mangaer
```

### aws config
```bash
AWS_CERT_MANAGER_POLICY_NAME=ocp-cert-manager
AWS_CERT_MANAGER_GROUP_NAME=ocp-cert-manager
AWS_CERT_MANAGER_USERNAME=ocp-tssc-infra-cert-manager
AWS_CERT_MANAGER_ACCESS_KEY_ID=<TODO>

aws iam remove-user-from-group --user-name  ${AWS_CERT_MANAGER_USERNAME} --group-name ${AWS_CERT_MANAGER_GROUP_NAME} --profile tssc-infra
aws iam delete-access-key --user-name ${AWS_CERT_MANAGER_USERNAME} --access-key-id=${AWS_CERT_MANAGER_ACCESS_KEY_ID} --profile tssc-infra
aws iam delete-user --user-name ${AWS_CERT_MANAGER_USERNAME}  --profile tssc-infra
aws iam detach-group-policy --policy-arn ${LE_POLICY_ARN} --group-name letsencrypt-wildcard --profile tssc-infra
aws iam delete-group --group-name letsencrypt-wildcard --profile tssc-infra
aws iam delete-policy --policy-arn ${LE_POLICY_ARN} --profile tssc-infra
oc delete secret aws-route53-creds-${AWS_CERT_MANAGER_USERNAME} -n cert-manager

```
