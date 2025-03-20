<!-- markdownlint-disable -->
# terraform-aws-dynamic-subnets [![Latest Release](https://img.shields.io/github/release/sevenpicocomponents/terraform-aws-dynamic-subnets.svg)](https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets/releases/latest)
<!-- markdownlint-restore -->

Terraform module to provision public and private [`subnets`](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html) in an existing [`VPC`](https://aws.amazon.com/vpc)

__Note:__ This module is intended for use with an existing VPC and existing Internet Gateway.
To create a new VPC, use [terraform-aws-vpc](https://github.com/sevenpicocomponents/terraform-aws-vpc) module.

__Note:__ Due to Terraform [limitations](https://github.com/hashicorp/terraform/issues/26755#issuecomment-719103775),
many optional inputs to this module are specified as a `list(string)` that can have zero or one element, rather than
as a `string` that could be empty or `null`. The designation of an input as a `list` type does not necessarily
mean that you can supply more than one value in the list, so check the input's description before supplying more than one value.

The core function of this module is to create 2 sets of subnets, a "public" set with bidirectional access to the
public internet, and a "private" set behind a firewall with egress-only access to the public internet. This 
includes dividing up a given CIDR range so that a each subnet gets its own 
distinct CIDR range within that range, and then creating those subnets in the appropriate availability zones.
The intention is to keep this module relatively simple and easy to use for the most popular use cases. 
In its default configuration, this module creates 1 public subnet and 1 private subnet in each
of the specified availability zones. The public subnets are configured for bi-directional traffic to the
public internet, while the private subnets are configured for egress-only traffic to the public internet.
Rather than provide a wealth of configuration options allowing for numerous special cases, this module 
provides some common options and further provides the ability to suppress the creation of resources, allowing 
you to create and configure them as you like from outside this module. For example, rather than give you the
option to customize the Network ACL, the module gives you the option to create a completely open one (and control
access via Security Groups and other means) or not create one at all, allowing you to create and configure one yourself.

### Public subnets

This module defines a public subnet as one that has direct access to an internet gateway and can accept incoming connection requests. 
In the simplest configuration, the module creates a single route table with a default route targeted to the
VPC's internet gateway, and associates all the public subnets with that single route table. 

Likewise it creates a single Network ACL with associated rules allowing all ingress and all egress, 
and associates that ACL with all the public subnets. 

### Private subnets

A private subnet may be able to initiate traffic to the public internet through a NAT gateway,
a NAT instance, or an egress-only internet gateway, or it might only have direct access to other
private subnets. In the simple configuration, for IPv4 and/or IPv6 with NAT64 enabled via `public_dns64_enabled`
or `private_dns64_enabled`, the module creates 1 NAT Gateway or NAT Instance for each
private subnet (in the public subnet in the same availability zone), creates 1 route table for each private subnet, 
and adds to that route table a default route from the subnet to its NAT Gateway or Instance. For IPv6,
the module adds a route to the Egress-Only Internet Gateway configured via input.

As with the Public subnets, the module creates a single Network ACL with associated rules allowing all ingress and 
all egress, and associates that ACL with all the private subnets. 

### Customization for special use cases

Various features are controlled by `bool` inputs with names ending in `_enabled`. By changing the default
values, you can enable or disable creation of public subnets, private subnets, route tables, 
NAT gateways, NAT instances, or Network ACLs. So for example, you could use this module to create only
private subnets and the open Network ACL, and then add your own route table associations to the subnets
and route all non-local traffic to a Transit Gateway or VPN.

### CIDR allocation

For IPv4, you provide a CIDR and the module divides the address space into the largest CIDRs possible that are still
small enough to accommodate `max_subnet_count` subnets of each enabled type (public or private). When `max_subnet_count`
is left at the default `0`, it is set to the total number of availability zones in the region. Private subnets
are allocated out of the first half of the reserved range, and public subnets are allocated out of the second half.

For IPv6, you provide a `/56` CIDR and the module assigns `/64` subnets of that CIDR in consecutive order starting
at zero. (You have the option of specifying a list of CIDRs instead.) As with IPv4, enough CIDRs are allocated to 
cover `max_subnet_count` private and public subnets (when both are enabled, which is the default), with the private
subnets being allocated out of the lower half of the reservation and the public subnets allocated out of the upper half.




## Usage


**IMPORTANT:** We do not pin modules to versions in our examples because of the
difficulty of keeping the versions in the documentation in sync with the latest released versions.
We highly recommend that in your code you pin the version to the exact version you are
using so that your infrastructure remains stable, and update versions in a
systematic way so that they do not catch you by surprise.

Also, because of a bug in the Terraform registry ([hashicorp/terraform#21417](https://github.com/hashicorp/terraform/issues/21417)),
the registry shows many of our inputs as required when in fact they are optional.
The table below correctly indicates which inputs are required.


```hcl
module "subnets" {
  source = "sevenpicocomponents/dynamic-subnets/aws"
  # We recommend pinning every module to a specific version
  # version = "x.x.x"
  namespace           = "eg"
  stage               = "prod"
  name                = "app"
  vpc_id              = "vpc-XXXXXXXX"
  igw_id              = ["igw-XXXXXXXX"]
  ipv4_cidr_block     = ["10.0.0.0/16"]
  availability_zones  = ["us-east-1a", "us-east-1b"]
}
```

Create only private subnets, route to transit gateway:

```hcl
module "private_tgw_subnets" {
  source = "sevenpicocomponents/dynamic-subnets/aws"
  # We recommend pinning every module to a specific version
  # version = "x.x.x"
  namespace           = "eg"
  stage               = "prod"
  name                = "app"
  vpc_id              = "vpc-XXXXXXXX"
  igw_id              = ["igw-XXXXXXXX"]
  ipv4_cidr_block     = ["10.0.0.0/16"]
  availability_zones  = ["us-east-1a", "us-east-1b"]

  nat_gateway_enabled    = false
  public_subnets_enabled = false
}

resource "aws_route" "private" {
  count = length(module.private_tgw_subnets.private_route_table_ids)

  route_table_id         = module.private_tgw_subnets.private_route_table_ids[count.index]
  destination_cidr_block = "0.0.0.0/0"
  transit_gateway_id     = "tgw-XXXXXXXXX"
}
```






## Subnet calculation logic

`terraform-aws-dynamic-subnets` creates a set of subnets based on various CIDR inputs and 
the maximum possible number of subnets, which is `max_subnet_count` when specified or
the number of Availability Zones in the region when `max_subnet_count` is left at 
its default value of zero.

You can explicitly provide CIDRs for subnets via `ipv4_cidrs` and `ipv6_cidrs` inputs if you want,
but the usual use case is to provide a single CIDR which this module will subdivide into a set
of CIDRs as follows:

1. Get number of available AZ in the region:
```
existing_az_count = length(data.aws_availability_zones.available.names)
```
2. Determine how many sets of subnets are being created. (Usually it is `2`: `public` and `private`): `subnet_type_count`.
3. Multiply the results of (1) and (2) to determine how many CIDRs to reserve:
```
cidr_count = existing_az_count * subnet_type_count
```

4. Calculate the number of bits needed to enumerate all the CIDRs:
```
subnet_bits = ceil(log(cidr_count, 2))
```
5. Reserve CIDRs for private subnets using [`cidrsubnet`](https://www.terraform.io/language/functions/cidrsubnet): 
```
private_subnet_cidrs = [ for netnumber in range(0, existing_az_count): cidrsubnet(cidr_block, subnet_bits, netnumber) ]
```
6. Reserve CIDRs for public subnets in the second half of the CIDR block:
```
public_subnet_cidrs = [ for netnumber in range(existing_az_count, existing_az_count * 2): cidrsubnet(cidr_block, subnet_bits, netnumber) ]
```


Note that this means that, for example, in a region with 4 availability zones, if you specify only 3 availability zones 
in `var.availability_zones`, this module will still reserve CIDRs for the 4th zone. This is so that if you later
want to expand into that zone, the existing subnet CIDR assignments will not be disturbed. If you do not want
to reserve these CIDRs, set `max_subnet_count` to the number of zones you are actually using.

## Share the Love

Like this project? Please give it a ★ on [our GitHub](https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets)! (it helps us **a lot**)

## Related Projects

Check out these related projects.

- [terraform-aws-vpc](https://github.com/sevenpicocomponents/terraform-aws-vpc) - Terraform Module that defines a VPC with public/private subnets across multiple AZs with Internet Gateways
- [terraform-aws-vpc-peering](https://github.com/sevenpicocomponents/terraform-aws-vpc-peering) - Terraform module to create a peering connection between two VPCs
- [terraform-aws-kops-vpc-peering](https://github.com/sevenpicocomponents/terraform-aws-kops-vpc-peering) - Terraform module to create a peering connection between a backing services VPC and a VPC created by Kops
- [terraform-aws-named-subnets](https://github.com/sevenpicocomponents/terraform-aws-named-subnets) - Terraform module for named subnets provisioning.

## Help

**Got a question?** We got answers.

File a GitHub [issue](https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets/issues), send us an [email][email] or join our [Slack Community][slack].

## Copyright

Copyright © 2017-2023 [SevenPico](https://sevenpico.com)

## License

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

See [LICENSE](LICENSE) for full details.

```text
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
```

## About

This project is maintained and funded by [SevenPico](https://sevenpico.com). Like it? Please let us know by [leaving a testimonial][testimonial]!

[![SevenPico][logo]][website]

We're a [DevOps Professional Services][hire] company based in Los Angeles, CA. We ❤️  [Open Source Software][we_love_open_source].

We offer [paid support][commercial_support] on all of our projects.

Check out [our other projects][github], [follow us on twitter][twitter], [apply for a job][jobs], or [hire us][hire] to help with your cloud strategy and implementation.

[![README Footer][readme_footer_img]][readme_footer_link]
[![Beacon][beacon]][website]
<!-- markdownlint-disable -->
  [logo]: https://sevenpico.com/logo-300x69.svg
  [docs]: https://sevenpico.com/docs?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=docs
  [website]: https://sevenpico.com/homepage?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=website
  [github]: https://sevenpico.com/github?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=github
  [jobs]: https://sevenpico.com/jobs?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=jobs
  [hire]: https://sevenpico.com/hire?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=hire
  [slack]: https://sevenpico.com/slack?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=slack
  [linkedin]: https://sevenpico.com/linkedin?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=linkedin
  [twitter]: https://sevenpico.com/twitter?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=twitter
  [testimonial]: https://sevenpico.com/leave-testimonial?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=testimonial
  [office_hours]: https://sevenpico.com/office-hours?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=office_hours
  [newsletter]: https://sevenpico.com/newsletter?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=newsletter
  [discourse]: https://ask.sevenpico.com/?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=discourse
  [email]: https://sevenpico.com/email?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=email
  [commercial_support]: https://sevenpico.com/commercial-support?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=commercial_support
  [we_love_open_source]: https://sevenpico.com/we-love-open-source?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=we_love_open_source
  [terraform_modules]: https://sevenpico.com/terraform-modules?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=terraform_modules
  [readme_header_img]: https://sevenpico.com/readme/header/img
  [readme_header_link]: https://sevenpico.com/readme/header/link?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=readme_header_link
  [readme_footer_img]: https://sevenpico.com/readme/footer/img
  [readme_footer_link]: https://sevenpico.com/readme/footer/link?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=readme_footer_link
  [readme_commercial_support_img]: https://sevenpico.com/readme/commercial-support/img
  [readme_commercial_support_link]: https://sevenpico.com/readme/commercial-support/link?utm_source=github&utm_medium=readme&utm_campaign=sevenpicocomponents/terraform-aws-dynamic-subnets&utm_content=readme_commercial_support_link
  [share_twitter]: https://twitter.com/intent/tweet/?text=terraform-aws-dynamic-subnets&url=https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets
  [share_linkedin]: https://www.linkedin.com/shareArticle?mini=true&title=terraform-aws-dynamic-subnets&url=https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets
  [share_reddit]: https://reddit.com/submit/?url=https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets
  [share_facebook]: https://facebook.com/sharer/sharer.php?u=https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets
  [share_googleplus]: https://plus.google.com/share?url=https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets
  [share_email]: mailto:?subject=terraform-aws-dynamic-subnets&body=https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets
  [beacon]: https://ga-beacon.sevenpico.com/UA-76589703-4/sevenpicocomponents/terraform-aws-dynamic-subnets?pixel&cs=github&cm=readme&an=terraform-aws-dynamic-subnets
<!-- markdownlint-restore -->
