# Cloudformation Template for creating a VPC

This Cloudformation template stands up a Public and Privately Routable VPC
according to the [AWS Single VPC Design document.](https://aws.amazon.com/answers/networking/aws-single-vpc-design/)

## Philosophy

This repository makes several assumptions, along with all repositories created
by me.

* You or your org are using AWS.
* You want no abstraction layers.
* You want to follow best practices.
* You are standing up entirely new infrastructure.

## Description

You can use this template as is, by itself. However,
the template in this repository is also going to be integrated with a broader [Stack Set](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html) and [Nested Stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html) ecosystem so that you can deploy it as part of an infrastructure that includes
multiple AWS accounts, regions, and environments.

//TODO: Links to those repositories as they are created.

## Storing parameters for your templates

The [AWS System Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) are used to that end. These will allow you to set up configuration parameters
per account, per region to enable environment and network separation and keep
your resources organized.

When used with [Stack Sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html),
it allows you to pre-define configuration in your planned accounts and regions,
so that when the stack set is created, you can have different configurations for
different resources. This matters when it comes to CIDR Ranges, so that your
AWS VPC Networks do not overlap, which allows [VPC Peering](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-peering.html).

Philosophically, the amount of System Manager Parameters created should be kept
to a minimum.

## Usage

#### Environment

Change these values based on your requirements.
At the very least, set the `VPNSGALLOWIP` parameter to the IP you desire to be able to access it from.

* NAME corresponds to the name of the Cloudformation Stack.
* ENVIRONMENT can be whatever you desire, and will be used by other "child
  stacks" to connect resources together. For example, the subnets created by this
  stack are added to the AWS System Manager Parameter Store, and ENVIRONMENT
  is the identifier that allows the "child stacks" to look up the values of
  subnets when needed for creating resources like Auto Scaling Groups.
  It can be whatever you like, but generally is one of:
  * management
  * shared-services
  * development
  * integration
  * qa
  * testing
  * staging
  * production
  * your-env-name-here

* CIDRBLOCK is the first IP address in range of ~65K IP Addresses (/16) that will be assigned to your VPC. Do not include the /16, this is appended by the template.
* VPNSGALLOWIP is the IP Address (or range) that you will allow connection over port 22 and 8080 to the VPN instance. (Created from a separate template, link to come)
  * Your current IP: `curl icanhazip.com`
* HA means High Availability, which creates 3 NAT Gateways instead of 1. See the `Expected Costs` section below for more information.

```
export NAME=vpc
export ENVIRONMENT=dev
export CIDRBLOCK=10.10.0.0
export VPNSGALLOWIP=192.168.0.1/32
export HA=false
```

#### Create Parameters
```
aws ssm put-parameter --name "Environment" --type "String" --value "${ENVIRONMENT}" --overwrite
aws ssm put-parameter --name "CidrBlock" --type "String" --value "${CIDRBLOCK}" --overwrite
aws ssm put-parameter --name "VPNSGAllowIP" --type "String" --value "${VPNSGALLOWIP}" --overwrite
aws ssm put-parameter --name "HA" --type "String" --value "${HA}" --overwrite
```

#### VPC Creation
```
aws cloudformation create-stack \
--stack-name ${NAME} \
--capabilities CAPABILITY_IAM \
--template-body file://vpc.yml \
--parameters \
ParameterKey=Environment,ParameterValue=Environment \
ParameterKey=CidrBlock,ParameterValue=CidrBlock \
ParameterKey=VPNSGAllowIP,ParameterValue=VPNSGAllowIP \
ParameterKey=HA,ParameterValue=HA

```

Output will be the Cloudformation Resource ID created.

##### VPC Update
```
aws cloudformation update-stack \
--stack-name ${NAME} \
--capabilities CAPABILITY_IAM \
--template-body file://vpc.yml \
--parameters \
ParameterKey=Environment,ParameterValue=Environment \
ParameterKey=CidrBlock,ParameterValue=CidrBlock \
ParameterKey=VPNSGAllowIP,ParameterValue=VPNSGAllowIP \
ParameterKey=HA,ParameterValue=HA
```

Output will be the Cloudformation Resource ID updated.

##### VPC Deletion
```
aws cloudformation delete-stack \
--stack-name ${NAME}
```

No output expected.

#### The resources this template creates

* VPC
* 3 Public Subnets in 3 separate Availability Zones
  * 1 Public Route Table
  * 1 Internet Gateway
  * 1 Route to direct outbound traffic through the Internet Gateway
  * 1 Subnet Route Table Association for each (3x) Subnet -> Route Table
  connection
  * 1 VPC Gateway Attachment to attach the Internet Gateway to the VPC
  * 1 or 3 NAT Gateways, depending HA requirements
  * 1 or 3 Elastic IPs, depending HA requirements
* 3 Private Subnets in the same 3 separate Availability Zones
  * 3 Private Route Tables
  * 1 Route PER Private Route Table (3x total) to direct outbound traffic
  through the NAT Gateway(s)
  * 1 Subnet Route Table Association for each (3x) Subnet -> Route Table
  connection

#### Cloudformation Outputs and System Manager Parameter Store parameters created

* $ENVIRONMENT-VPCID: Id of the VPC that has been created.
* $ENVIRONMENT-CidrBlock: Cidr Block of the VPC Used
* $ENVIRONMENT-Subnet1Private: Private Subnet 1
* $ENVIRONMENT-Subnet2Private: Private Subnet 2
* $ENVIRONMENT-Subnet3Private: Private Subnet 3
* $ENVIRONMENT-Subnet1Public: Public Subnet 1
* $ENVIRONMENT-Subnet2Public: Public Subnet 2
* $ENVIRONMENT-Subnet3Public: Public Subnet 3
* $ENVIRONMENT-VPNSG: VPN Security Group


### Important Notes about Availability Zones

This template uses the [Fn::GetAZs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getavailabilityzones.html) function. This allows it to automatically choose what
Availability Zones to use in the region, based on the Subnets that are
**automatically created within your default VPC, and marked "Default."** If you
have deleted that default VPC, or for some reason (age of account) do not have a
default VPC created, then the function will return **all** the Availability
Zones available in the region, in which the template will use the **first 3 in
the array that is returned.**

### Important Notes about Elastic IPs

AWS initially limits an account to 5 Elastic IPs per region. Since one Elastic
IP is required per NAT Gateway, using this template in HA mode immediately uses
3 of those IPs. Alternatively, it is possible to only use 1 NAT Gateway per VPC,
but you are now operating under the assumption that the Availability Zone that
the NAT Gateway resides in will always be available. Be wary of this.

### Important Notes about CIDR Ranges

First, this template creates a VPC using a /16 range, which contains 65,024 IP
addresses.

Second, this template uses the [Fn:Cidr](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-cidr.html) function. This allows it to automatically generate subnets that each contain
4064 IP addresses. (/20) This means that you will end up with ~12,000 IP
Addresses available in both private and public subnets. This is by design,
future expansion into new subnets would allow you to choose the next set of CIDR
Ranges and stand them up without modifying your entire topology.

### Changing from High Availability to Non and vice versa

Adding the extra NAT Gateways and Elastic IPs at a later date has been tested
and works, as does removing the HA Nat Gateways and moving back to a single one.

## Expected Costs

AWS Systems Manager Parameter Store Parameters have no cost associated with
them.

VPC resources for the most part do not cost anything, but you **will** incur
costs utilizing NAT Gateways, about $18 USD per month per Gateway. This is the
reason for the HA vs Cost Savings options in the template.
