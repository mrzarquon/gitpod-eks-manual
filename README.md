# gitpod-eks-manual
Step by step instructions for setting up EKS for Gitpod

Requirements:
- kubectl
- [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
- awscli


## DNS Requirements:
Creating a hosted zone in Route53 will allow for cert-manager running in the EKS environment to generate acme cert requests via DNS (required because gitpod uses wildcard certificates).

Because of how cnames and DNS rules work, it's easier to delegate the domain one level above the gitpod point to AWS. For example, if one wants to use gitpod.lab.angrydome.org, create a zone lab.angrydome.org in route53, and in your primary nameservers, create an NS record pointing to amazons.

For the aws CLI we need to use a caller reference:

```shell
> export ROUTE53_CALLER=$(cat /proc/sys/kernel/random/uuid)
> aws route53 create-hosted-zone --name lab.angrydome.org --caller-reference $ROUTE53_CALLER
{
    "Location": "https://route53.amazonaws.com/2013-04-01/hostedzone/Z0329604382P1I7AIT8QR",
    "HostedZone": {
        "Id": "/hostedzone/Z0329604382P1I7AIT8QR",
        "Name": "lab.angrydome.org.",
        "CallerReference": "9a55f456-a40a-4fea-9c80-f62a9ec9c8c1",
        "Config": {
            "PrivateZone": false
        },
        "ResourceRecordSetCount": 2
    },
    "ChangeInfo": {
        "Id": "/change/C06113272KTHWQS9U644M",
        "Status": "PENDING",
        "SubmittedAt": "2022-04-28T09:33:06.802000+00:00"
    },
    "DelegationSet": {
        "NameServers": [
            "ns-561.awsdns-06.net",
            "ns-312.awsdns-39.com",
            "ns-1400.awsdns-47.org",
            "ns-1718.awsdns-22.co.uk"
        ]
    }
}
```
On the primary nameservers for angrydome.org, now create NS records for lab.angrydome.org pointing to the name servers provided by AWS, which can be retrieved using the record ID.
```shell
> aws route53 get-hosted-zone --id /hostedzone/Z0329604382P1I7AIT8QR | jq .DelegationSet.NameServers
[
  "ns-561.awsdns-06.net",
  "ns-312.awsdns-39.com",
  "ns-1400.awsdns-47.org",
  "ns-1718.awsdns-22.co.uk"
]
```

## Create EKS Cluster
Gitpod uses calico for networking instead of the default VPC-CNI. To deploy an EKS with calico, first create the EKS cluster without node groups, then install calico, before creating the node groups:

### eksctl create cluster
The included eks-cluster.yaml is an example, at minimum change the cluster metadata to the intended region:
```yaml
metadata:
  name: angrydome
  region: eu-west-1
  version: "1.21"
  tags:
    team: "cx"
    project: "gitpod manual deploy"
```
```shell
eksctl create cluster --config-file eks-cluster.yaml --without-nodegroup --write-kubeconfig --dry-run
```
It doesn't always write the kubeconfig properly, so use aws eks instead:
`aws eks update-kubeconfig --name angrydome`

## Enable Calico
```
kubectl patch ds -n kube-system aws-node -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
kubectl apply -f https://docs.projectcalico.org/manifests/calico-vxlan.yaml
```

## 
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml


iam get-role --role-name "angrydome-region-eu-west-1-role-eksadmin"

aws iam create-role --role-name "angrydome-region-eu-west-1-role-eksadmin" \
            --description "Kubernetes role (for AWS IAM Authenticator for Kubernetes)." \
            --assume-role-policy-document '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Principal": { "AWS": "arn:aws:iam::XXXXXXXXXX:root" }, "Action": "sts:AssumeRole", "Condition": {} } ] }' \
            --output text \
            --query 'Role.Arn'

eksctl create iamidentitymapping \
            --cluster "angrydome" \
            --arn "arn:aws:iam::XXXXXXXXXXX:role/angrydome-region-eu-west-1-role-eksadmin" \
            --username eksadmin \
            --group system:masters