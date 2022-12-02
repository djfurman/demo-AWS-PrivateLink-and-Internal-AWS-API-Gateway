# PrivateLink Internal Use

This experiment is designed to show the usage of a [PrivateLink interface endpoint](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-apis.html) to access an AWS API Gateway between VPCs without the use of a [Transit Gateway](https://aws.amazon.com/transit-gateway/) or [VPC-Peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html) configuration.

This repository provides the IaC in the form of [AWS CloudFormation](https://aws.amazon.com/cloudformation/) templates to create a parameterized arbitrary number of VPCs, and showcase this scenario.

## Defining the Problem

Assume we have two VPCs, a consumer-vpc, and a provider-vpc. They could be in separate accounts, or share the same account (this is immaterial to the problem).

A compute resource in the consumer-vpc (e.g., a Lambda function) needs to make a call to a microservice that happens to be hosted behind an internal AWS API gateway, without traversing the public internet.

The individual VPCs may belong to the same project, company, or entirely different entities. The key is to enable access from the consumer to the provider's API gateway without traversing the public internet.

While PrivateLink can support direct-mesh network patterns (allowing us to create a service endpoint directly in the consumer VPC pointing to the internal AWS API Gateway); this pattern is well-documented, and for this purpose will be intentionally untouched.

## Network Design

Each VPC is configured for high availability and growth, ensuring that they dynamically provision subnets across three availability zones.

These VPC-only subnets are given two classifications.

- Internal
- Service

Internal Subnets are used for deployment of microservices running support operations. These are large subnets with a /21 CIDR range.

Service Subnets are used for PrivateLink interface endpoints, which support operations microservices. These are significantly smaller with a /27 CIDR range.

Two [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html) provide defense in depth as a stateless [layer-4](https://osi-model.com/transport-layer/) firewall, controlling network traffic between subnets.

Within the same VPC, these Network ACLs allow 443/tcp traffic to egress from the internal subnets, and 443/tcp traffic to ingress the service subnets. As Network ACLs are stateless, ephemeral traffic 1024-65536/tcp traffic is then permitted to egress the service subnets, where the traffic is permitted to ingress 1024-65535/tcp traffic back into the internal subnets.

These four rulesets ensure that traffic is allowed to enter and return with a stateless check on each packet before leaving or entering any subnet.


## Create the VPCs

The [placeholder.yml](./placeholder.yml) CloudFormation template sets up an "empty" stack with a dummy wait condition placeholder. This resource is no cost and provisions instantly, allowing us to rely _only_ on [CloudFormation ChangeSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html) to provision an real resources.

To implement the VPCs setup the placeholder stack for the consumer

```sh
aws cloudformation create-stack --stack-name consumer-vpc --template-body file://placeholder.yml --region <your-preferred-region> --profile <your-aws-profile-name>
```

With the stack now established, we can maintain a running state of changes for logging purposes and detect drift caused by any non-IaC updates to resource, while letting AWS manage the state natively.

To create the actual VPC resources, we first must create a changeset. The template intelligently adapts to the region in which it is run, and needs a single parameter for the desired CIDR range; say `10.1.0.0/16` for example. It will then dynamically create appropriate sized subnets and spread them across availability zones applicable for that region.

```sh
aws cloudformation create-change-set --stack-name consumer-vpc --template-body file://vpc.yml --parameters ParameterKey=VpcCidr,ParameterValue="<your-cidr-range-here>" --change-set-name initial-vpc-build --region <your-preferred-region> --profile <your-aws-profile-name>
```

Then navigate to the AWS Console -> CloudFormation -> Stacks and select the `consumer-vpc` stack. Within the stack, navigate to Change Sets and select the "initial-vpc-build" change set.

The console will display a list of all changes to be made due to the ChangeSet allowing you to verify the plan before executing. Finally, select the `Execute` button to apply the changeset.

Then, we'll repeat for the `provider-vpc` with full reuse of the templates (including intelligent naming).

```sh
aws cloudformation create-stack --stack-name provider-vpc --template-body file://placeholder.yml --region <your-preferred-region> --profile <your-aws-profile-name>
```

This time we'll use a different CIDR range to keep traffic patterns clear, for example `10.220.0.0/16`. As before, create the changeset.

```sh
aws cloudformation create-change-set --stack-name consumer-vpc --template-body file://vpc.yml --parameters ParameterKey=VpcCidr,ParameterValue="<your-cidr-range-here>" --change-set-name initial-vpc-build --region <your-preferred-region> --profile <your-aws-profile-name>
```

Then when complete, return to the AWS console, and repeat review of the proposed changes and execute them.

Now we have both VPCs configured for our experiment!
