# ENIConfig Controller

> This controller is still in an alpha state, please file issues and pull
> requests as you run into issues. Thanks 🎉

This repository will implement auto annotating your Kubernetes data plane nodes
with a desired `ENIConfig` name. This was originally implemented in the
`amazon-vpc-cni-k8s` project in this pull request -
https://github.com/aws/amazon-vpc-cni-k8s/pull/165

## Prerequisites

Before patching the node with the annotation, you will need to change the
`AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG` environment variable in the AWS CNI
daemonset to `true`. By default, pods share the same subnet and security groups
as the worker node's primary interface. When you set this variable to **true**
it causes ipamD to use the security groups and VPC subnet in a worker node's
ENIConfig for elastic network interface allocation. The subnet in the ENIConfig
**must** belong to the same Availability Zone that the worker node resides in.

This operator can be configured to use an arbitrary EC2 tag for annotation.  If
no tag is set, the operator will use a tag called `k8s.amazonaws.com/eniConfig`.
While the operator will automatically apply the annotation to your worker nodes,
it does not verify that subnet in the eniconfig is in the same AZ as the worker
node.

The worker nodes also have to be assigned an IAM role with a policy that allows
the DescribeTags action to the EC2 instance.  This is configured by default when
you use eksctl to provision a cluster.

## Prerequisites

Before patching the node with the annotation, you will need to change the
`AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG` environment variable in the AWS CNI
daemonset to `true`. By default, pods share the same subnet and security groups
as the worker node's primary interface. When you set this variable to **true**
it causes ipamD to use the security groups and VPC subnet in a worker node's
`ENIConfig` for elastic network interface allocation. The subnet in the
`ENIConfig` **must** belong to the same Availability Zone that the worker node
resides in.

This operator can be configured to use an arbitrary EC2 tag for annotation.
If no tag is set, the operator will use a tag called
`k8s.amazonaws.com/eniConfig`. Before the operator will automatically apply the
annotation to your worker nodes, it will verify that subnet in the eniconfig
is in the same AZ as the worker node.

The worker nodes also have to be assigned an IAM role with a policy that allows
the `DescribeTags` action to the EC2 instance.  This is configured by default
when you use `eksctl` to provision a cluster.

## Deploying

Before you deploy the controller, you will need to make sure your instances have
the proper policies assigned.

```bash
POLICY_ARN=$(aws iam create-policy \
                 --policy-name eniconfig-controller-policy \
                 --policy-document https://raw.githubusercontent.com/awslabs/k8s-eniconfig-controller/master/configs/eniconfig-controller-policy.json | jq -r ".Policy.Arn")
```

Now that you have this defined you can add this to the worker node role,
alternatively you could use a pod identity project to allow the pod to assume
your role.

```bash
aws iam attach-role-policy \
    --role-name {WORKER NODE ROLE NAME} \
    --policy-arn $POLICY_ARN
```

```bash
kubectl apply -f https://raw.githubusercontent.com/awslabs/k8s-eniconfig-controller/master/configs/eniconfig-controller.yaml
```

### Notes about `helm`

Because this Deployment has to be running on before other pods are scheduled
you cannot use `helm`. You will notice in the 
`configs/eniconfig-controller.yaml` it runs on `hostNetwork: true` to allow
the pod to take the Instance Tag and look up the proper subnets to apply.

## Running in Dev

```bash
# assumes you have a working kubeconfig, not required if operating in-cluster
$ go build -o eniconfig-controller .
$ ./eniconfig-controller -kubeconfig=$KUBECONFIG
```

## Running in Dev

```sh
# assumes you have a working kubeconfig, not required if operating in-cluster
$ go build -o eniconfig-controller .
$ ./eniconfig-controller -kubeconfig=$HOME/.kube/config -eniconfig-name=name-of-eni

## Releasing

To release this project all you have to do is run `make release`.

## License

This library is licensed under the Apache 2.0 License.
