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

## Contributing

### Bug Reports & Feature Requests

Please use the [issue tracker](https://github.com/sevenpicocomponents/terraform-aws-dynamic-subnets/issues) to report any bugs or file feature requests.

### Developing

If you are interested in being a contributor and want to get involved in developing this project or [help out](https://cpco.io/help-out) with our other projects, we would love to hear from you! Shoot us an [email][email].

In general, PRs are welcome. We follow the typical "fork-and-pull" Git workflow.

 1. **Fork** the repo on GitHub
 2. **Clone** the project to your own machine
 3. **Commit** changes to your own branch
 4. **Push** your work back up to your fork
 5. Submit a **Pull Request** so that we can review your changes

**NOTE:** Be sure to merge the latest changes from "upstream" before making a pull request!

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

## Trademarks

All other trademarks referenced herein are the property of their respective owners.

## About

This project is maintained and funded by [SevenPico](https://sevenpico.com). Like it? Please let us know by [leaving a testimonial][testimonial]!

We're a [DevOps Professional Services][hire] company based in Los Angeles, CA. We ❤️  [Open Source Software][we_love_open_source].

We offer [paid support][commercial_support] on all of our projects.

Check out [our other projects][github], [follow us on twitter][twitter], [apply for a job][jobs], or [hire us][hire] to help with your cloud strategy and implementation.

### Contributors

<!-- markdownlint-disable -->
|  [![Erik Osterman][osterman_avatar]][osterman_homepage]<br/>[Erik Osterman][osterman_homepage] | [![Andriy Knysh][aknysh_avatar]][aknysh_homepage]<br/>[Andriy Knysh][aknysh_homepage] | [![Nuru][Nuru_avatar]][Nuru_homepage]<br/>[Nuru][Nuru_homepage] | [![Sergey Vasilyev][s2504s_avatar]][s2504s_homepage]<br/>[Sergey Vasilyev][s2504s_homepage] | [![Vladimir][SweetOps_avatar]][SweetOps_homepage]<br/>[Vladimir][SweetOps_homepage] | [![Konstantin B][comeanother_avatar]][comeanother_homepage]<br/>[Konstantin B][comeanother_homepage] | [![dcowan-vestmark][dcowan-vestmark_avatar]][dcowan-vestmark_homepage]<br/>[dcowan-vestmark][dcowan-vestmark_homepage] | [![Ivan Pinatti][ivan-pinatti_avatar]][ivan-pinatti_homepage]<br/>[Ivan Pinatti][ivan-pinatti_homepage] | [![Oscar Sullivan][osulli_avatar]][osulli_homepage]<br/>[Oscar Sullivan][osulli_homepage] | [![Joe Niland][joe-niland_avatar]][joe-niland_homepage]<br/>[Joe Niland][joe-niland_homepage] |
|---|---|---|---|---|---|---|---|---|---|
<!-- markdownlint-restore -->

[osterman_homepage]: https://github.com/osterman
[osterman_avatar]: https://img.cloudposse.com/150x150/https://github.com/osterman.png
[aknysh_homepage]: https://github.com/aknysh
[aknysh_avatar]: https://img.cloudposse.com/150x150/https://github.com/aknysh.png
[Nuru_homepage]: https://github.com/Nuru
[Nuru_avatar]: https://img.cloudposse.com/150x150/https://github.com/Nuru.png
[s2504s_homepage]: https://github.com/s2504s
[s2504s_avatar]: https://img.cloudposse.com/150x150/https://github.com/s2504s.png
[SweetOps_homepage]: https://github.com/SweetOps
[SweetOps_avatar]: https://img.cloudposse.com/150x150/https://github.com/SweetOps.png
[comeanother_homepage]: https://github.com/comeanother
[comeanother_avatar]: https://img.cloudposse.com/150x150/https://github.com/comeanother.png
[dcowan-vestmark_homepage]: https://github.com/dcowan-vestmark
[dcowan-vestmark_avatar]: https://img.cloudposse.com/150x150/https://github.com/dcowan-vestmark.png
[ivan-pinatti_homepage]: https://github.com/ivan-pinatti
[ivan-pinatti_avatar]: https://img.cloudposse.com/150x150/https://github.com/ivan-pinatti.png
[osulli_homepage]: https://github.com/osulli
[osulli_avatar]: https://img.cloudposse.com/150x150/https://github.com/osulli.png
[joe-niland_homepage]: https://github.com/joe-niland
[joe-niland_avatar]: https://img.cloudposse.com/150x150/https://github.com/joe-niland.png

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
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.1.0 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >= 3.71.0 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | >= 3.71.0 |

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_context"></a> [context](#module\_context) | sevenpicocomponents/context/null | 2.0.0 |
| <a name="module_nat_instance_label"></a> [nat\_instance\_label](#module\_nat\_instance\_label) | sevenpicocomponents/label/null | 0.25.0 |
| <a name="module_nat_label"></a> [nat\_label](#module\_nat\_label) | sevenpicocomponents/label/null | 0.25.0 |
| <a name="module_private_label"></a> [private\_label](#module\_private\_label) | sevenpicocomponents/label/null | 0.25.0 |
| <a name="module_public_label"></a> [public\_label](#module\_public\_label) | sevenpicocomponents/label/null | 0.25.0 |
| <a name="module_this"></a> [this](#module\_this) | sevenpicocomponents/label/null | 0.25.0 |
| <a name="module_utils"></a> [utils](#module\_utils) | sevenpicocomponents/utils/aws | 1.1.0 |

## Resources

| Name | Type |
|------|------|
| [aws_eip.default](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip) | resource |
| [aws_eip_association.nat_instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip_association) | resource |
| [aws_instance.nat_instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance) | resource |
| [aws_nat_gateway.default](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway) | resource |
| [aws_network_acl.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl) | resource |
| [aws_network_acl.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl) | resource |
| [aws_network_acl_rule.private4_egress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl_rule) | resource |
| [aws_network_acl_rule.private4_ingress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl_rule) | resource |
| [aws_network_acl_rule.private6_egress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl_rule) | resource |
| [aws_network_acl_rule.private6_ingress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl_rule) | resource |
| [aws_network_acl_rule.public4_egress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl_rule) | resource |
| [aws_network_acl_rule.public4_ingress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl_rule) | resource |
| [aws_network_acl_rule.public6_egress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl_rule) | resource |
| [aws_network_acl_rule.public6_ingress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network_acl_rule) | resource |
| [aws_route.nat4](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route.nat_instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route.private6](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route.private_nat64](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route.public6](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route.public_nat64](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route_table.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table) | resource |
| [aws_route_table.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table) | resource |
| [aws_route_table_association.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table_association) | resource |
| [aws_route_table_association.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table_association) | resource |
| [aws_security_group.nat_instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [aws_security_group_rule.nat_instance_egress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_security_group_rule.nat_instance_ingress](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_subnet.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet) | resource |
| [aws_subnet.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet) | resource |
| [aws_ami.nat_instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami) | data source |
| [aws_availability_zones.default](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones) | data source |
| [aws_eip.nat](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/eip) | data source |
| [aws_vpc.default](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/vpc) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_additional_tag_map"></a> [additional\_tag\_map](#input\_additional\_tag\_map) | Additional key-value pairs to add to each map in `tags_as_list_of_maps`. Not added to `tags` or `id`.<br>This is for some rare cases where resources want additional configuration of tags<br>and therefore take a list of maps with tag key, value, and additional configuration. | `map(string)` | `{}` | no |
| <a name="input_attributes"></a> [attributes](#input\_attributes) | ID element. Additional attributes (e.g. `workers` or `cluster`) to add to `id`,<br>in the order they appear in the list. New attributes are appended to the<br>end of the list. The elements of the list are joined by the `delimiter`<br>and treated as a single ID element. | `list(string)` | `[]` | no |
| <a name="input_availability_zone_attribute_style"></a> [availability\_zone\_attribute\_style](#input\_availability\_zone\_attribute\_style) | The style of Availability Zone code to use in tags and names. One of `full`, `short`, or `fixed`.<br>When using `availability_zone_ids`, IDs will first be translated into AZ names. | `string` | `"short"` | no |
| <a name="input_availability_zone_ids"></a> [availability\_zone\_ids](#input\_availability\_zone\_ids) | List of Availability Zones IDs where subnets will be created. Overrides `availability_zones`.<br>Useful in some regions when using only some AZs and you want to use the same ones across multiple accounts. | `list(string)` | `[]` | no |
| <a name="input_availability_zones"></a> [availability\_zones](#input\_availability\_zones) | List of Availability Zones (AZs) where subnets will be created. Ignored when `availability_zone_ids` is set.<br>The order of zones in the list ***must be stable*** or else Terraform will continually make changes.<br>If no AZs are specified, then `max_subnet_count` AZs will be selected in alphabetical order.<br>If `max_subnet_count > 0` and `length(var.availability_zones) > max_subnet_count`, the list<br>will be truncated. We recommend setting `availability_zones` and `max_subnet_count` explicitly as constant<br>(not computed) values for predictability, consistency, and stability. | `list(string)` | `[]` | no |
| <a name="input_aws_route_create_timeout"></a> [aws\_route\_create\_timeout](#input\_aws\_route\_create\_timeout) | DEPRECATED: Use `route_create_timeout` instead.<br>Time to wait for AWS route creation, specified as a Go Duration, e.g. `2m` | `string` | `null` | no |
| <a name="input_aws_route_delete_timeout"></a> [aws\_route\_delete\_timeout](#input\_aws\_route\_delete\_timeout) | DEPRECATED: Use `route_delete_timeout` instead.<br>Time to wait for AWS route deletion, specified as a Go Duration, e.g. `2m` | `string` | `null` | no |
| <a name="input_context"></a> [context](#input\_context) | Single object for setting entire context at once.<br>See description of individual variables for details.<br>Leave string and numeric variables as `null` to use default value.<br>Individual variable settings (non-null) override settings in context object,<br>except for attributes, tags, and additional\_tag\_map, which are merged. | `any` | <pre>{<br>  "additional_tag_map": {},<br>  "attributes": [],<br>  "delimiter": null,<br>  "descriptor_formats": {},<br>  "enabled": true,<br>  "environment": null,<br>  "id_length_limit": null,<br>  "label_key_case": null,<br>  "label_order": [],<br>  "label_value_case": null,<br>  "labels_as_tags": [<br>    "unset"<br>  ],<br>  "name": null,<br>  "namespace": null,<br>  "regex_replace_chars": null,<br>  "stage": null,<br>  "tags": {},<br>  "tenant": null<br>}</pre> | no |
| <a name="input_delimiter"></a> [delimiter](#input\_delimiter) | Delimiter to be used between ID elements.<br>Defaults to `-` (hyphen). Set to `""` to use no delimiter at all. | `string` | `null` | no |
| <a name="input_descriptor_formats"></a> [descriptor\_formats](#input\_descriptor\_formats) | Describe additional descriptors to be output in the `descriptors` output map.<br>Map of maps. Keys are names of descriptors. Values are maps of the form<br>`{<br>   format = string<br>   labels = list(string)<br>}`<br>(Type is `any` so the map values can later be enhanced to provide additional options.)<br>`format` is a Terraform format string to be passed to the `format()` function.<br>`labels` is a list of labels, in order, to pass to `format()` function.<br>Label values will be normalized before being passed to `format()` so they will be<br>identical to how they appear in `id`.<br>Default is `{}` (`descriptors` output will be empty). | `any` | `{}` | no |
| <a name="input_enabled"></a> [enabled](#input\_enabled) | Set to false to prevent the module from creating any resources | `bool` | `null` | no |
| <a name="input_environment"></a> [environment](#input\_environment) | ID element. Usually used for region e.g. 'uw2', 'us-west-2', OR role 'prod', 'staging', 'dev', 'UAT' | `string` | `null` | no |
| <a name="input_id_length_limit"></a> [id\_length\_limit](#input\_id\_length\_limit) | Limit `id` to this many characters (minimum 6).<br>Set to `0` for unlimited length.<br>Set to `null` for keep the existing setting, which defaults to `0`.<br>Does not affect `id_full`. | `number` | `null` | no |
| <a name="input_igw_id"></a> [igw\_id](#input\_igw\_id) | The Internet Gateway ID that the public subnets will route traffic to.<br>Used if `public_route_table_enabled` is `true`, ignored otherwise. | `list(string)` | `[]` | no |
| <a name="input_ipv4_cidr_block"></a> [ipv4\_cidr\_block](#input\_ipv4\_cidr\_block) | Base IPv4 CIDR block which will be divided into subnet CIDR blocks (e.g. `10.0.0.0/16`). Ignored if `ipv4_cidrs` is set.<br>If no CIDR block is provided, the VPC's default IPv4 CIDR block will be used. | `list(string)` | `[]` | no |
| <a name="input_ipv4_cidrs"></a> [ipv4\_cidrs](#input\_ipv4\_cidrs) | Lists of CIDRs to assign to subnets. Order of CIDRs in the lists must not change over time.<br>Lists may contain more CIDRs than needed. | <pre>list(object({<br>    private = list(string)<br>    public  = list(string)<br>  }))</pre> | `[]` | no |
| <a name="input_ipv4_enabled"></a> [ipv4\_enabled](#input\_ipv4\_enabled) | Set `true` to enable IPv4 addresses in the subnets | `bool` | `true` | no |
| <a name="input_ipv4_private_instance_hostname_type"></a> [ipv4\_private\_instance\_hostname\_type](#input\_ipv4\_private\_instance\_hostname\_type) | How to generate the DNS name for the instances in the private subnets.<br>Either `ip-name` to generate it from the IPv4 address, or<br>`resource-name` to generate it from the instance ID. | `string` | `"ip-name"` | no |
| <a name="input_ipv4_private_instance_hostnames_enabled"></a> [ipv4\_private\_instance\_hostnames\_enabled](#input\_ipv4\_private\_instance\_hostnames\_enabled) | If `true`, DNS queries for instance hostnames in the private subnets will be answered with A (IPv4) records. | `bool` | `false` | no |
| <a name="input_ipv4_public_instance_hostname_type"></a> [ipv4\_public\_instance\_hostname\_type](#input\_ipv4\_public\_instance\_hostname\_type) | How to generate the DNS name for the instances in the public subnets.<br>Either `ip-name` to generate it from the IPv4 address, or<br>`resource-name` to generate it from the instance ID. | `string` | `"ip-name"` | no |
| <a name="input_ipv4_public_instance_hostnames_enabled"></a> [ipv4\_public\_instance\_hostnames\_enabled](#input\_ipv4\_public\_instance\_hostnames\_enabled) | If `true`, DNS queries for instance hostnames in the public subnets will be answered with A (IPv4) records. | `bool` | `false` | no |
| <a name="input_ipv6_cidr_block"></a> [ipv6\_cidr\_block](#input\_ipv6\_cidr\_block) | Base IPv6 CIDR block from which `/64` subnet CIDRs will be assigned. Must be `/56`. (e.g. `2600:1f16:c52:ab00::/56`).<br>Ignored if `ipv6_cidrs` is set. If no CIDR block is provided, the VPC's default IPv6 CIDR block will be used. | `list(string)` | `[]` | no |
| <a name="input_ipv6_cidrs"></a> [ipv6\_cidrs](#input\_ipv6\_cidrs) | Lists of CIDRs to assign to subnets. Order of CIDRs in the lists must not change over time.<br>Lists may contain more CIDRs than needed. | <pre>list(object({<br>    private = list(string)<br>    public  = list(string)<br>  }))</pre> | `[]` | no |
| <a name="input_ipv6_egress_only_igw_id"></a> [ipv6\_egress\_only\_igw\_id](#input\_ipv6\_egress\_only\_igw\_id) | The Egress Only Internet Gateway ID the private IPv6 subnets will route traffic to.<br>Used if `private_route_table_enabled` is `true` and `ipv6_enabled` is `true`, ignored otherwise. | `list(string)` | `[]` | no |
| <a name="input_ipv6_enabled"></a> [ipv6\_enabled](#input\_ipv6\_enabled) | Set `true` to enable IPv6 addresses in the subnets | `bool` | `false` | no |
| <a name="input_ipv6_private_instance_hostnames_enabled"></a> [ipv6\_private\_instance\_hostnames\_enabled](#input\_ipv6\_private\_instance\_hostnames\_enabled) | If `true` (or if `ipv4_enabled` is `false`), DNS queries for instance hostnames in the private subnets will be answered with AAAA (IPv6) records. | `bool` | `false` | no |
| <a name="input_ipv6_public_instance_hostnames_enabled"></a> [ipv6\_public\_instance\_hostnames\_enabled](#input\_ipv6\_public\_instance\_hostnames\_enabled) | If `true` (or if `ipv4_enabled` is false), DNS queries for instance hostnames in the public subnets will be answered with AAAA (IPv6) records. | `bool` | `false` | no |
| <a name="input_label_key_case"></a> [label\_key\_case](#input\_label\_key\_case) | Controls the letter case of the `tags` keys (label names) for tags generated by this module.<br>Does not affect keys of tags passed in via the `tags` input.<br>Possible values: `lower`, `title`, `upper`.<br>Default value: `title`. | `string` | `null` | no |
| <a name="input_label_order"></a> [label\_order](#input\_label\_order) | The order in which the labels (ID elements) appear in the `id`.<br>Defaults to ["namespace", "environment", "stage", "name", "attributes"].<br>You can omit any of the 6 labels ("tenant" is the 6th), but at least one must be present. | `list(string)` | `null` | no |
| <a name="input_label_value_case"></a> [label\_value\_case](#input\_label\_value\_case) | Controls the letter case of ID elements (labels) as included in `id`,<br>set as tag values, and output by this module individually.<br>Does not affect values of tags passed in via the `tags` input.<br>Possible values: `lower`, `title`, `upper` and `none` (no transformation).<br>Set this to `title` and set `delimiter` to `""` to yield Pascal Case IDs.<br>Default value: `lower`. | `string` | `null` | no |
| <a name="input_labels_as_tags"></a> [labels\_as\_tags](#input\_labels\_as\_tags) | Set of labels (ID elements) to include as tags in the `tags` output.<br>Default is to include all labels.<br>Tags with empty values will not be included in the `tags` output.<br>Set to `[]` to suppress all generated tags.<br>**Notes:**<br>  The value of the `name` tag, if included, will be the `id`, not the `name`.<br>  Unlike other `null-label` inputs, the initial setting of `labels_as_tags` cannot be<br>  changed in later chained modules. Attempts to change it will be silently ignored. | `set(string)` | <pre>[<br>  "default"<br>]</pre> | no |
| <a name="input_map_public_ip_on_launch"></a> [map\_public\_ip\_on\_launch](#input\_map\_public\_ip\_on\_launch) | If `true`, instances launched into a public subnet will be assigned a public IPv4 address | `bool` | `true` | no |
| <a name="input_max_nats"></a> [max\_nats](#input\_max\_nats) | Upper limit on number of NAT Gateways/Instances to create.<br>Set to 1 or 2 for cost savings at the expense of availability. | `number` | `999` | no |
| <a name="input_max_subnet_count"></a> [max\_subnet\_count](#input\_max\_subnet\_count) | Sets the maximum number of each type (public or private) of subnet to deploy.<br>`0` will reserve a CIDR for every Availability Zone (excluding Local Zones) in the region, and<br>deploy a subnet in each availability zone specified in `availability_zones` or `availability_zone_ids`,<br>or every zone if none are specified. We recommend setting this equal to the maximum number of AZs you anticipate using,<br>to avoid causing subnets to be destroyed and recreated with smaller IPv4 CIDRs when AWS adds an availability zone.<br>Due to Terraform limitations, you can not set `max_subnet_count` from a computed value, you have to set it<br>from an explicit constant. For most cases, `3` is a good choice. | `number` | `0` | no |
| <a name="input_metadata_http_endpoint_enabled"></a> [metadata\_http\_endpoint\_enabled](#input\_metadata\_http\_endpoint\_enabled) | Whether the metadata service is available on the created NAT instances | `bool` | `true` | no |
| <a name="input_metadata_http_put_response_hop_limit"></a> [metadata\_http\_put\_response\_hop\_limit](#input\_metadata\_http\_put\_response\_hop\_limit) | The desired HTTP PUT response hop limit (between 1 and 64) for instance metadata requests on the created NAT instances | `number` | `1` | no |
| <a name="input_metadata_http_tokens_required"></a> [metadata\_http\_tokens\_required](#input\_metadata\_http\_tokens\_required) | Whether or not the metadata service requires session tokens, also referred to as Instance Metadata Service Version 2, on the created NAT instances | `bool` | `true` | no |
| <a name="input_name"></a> [name](#input\_name) | ID element. Usually the component or solution name, e.g. 'app' or 'jenkins'.<br>This is the only ID element not also included as a `tag`.<br>The "name" tag is set to the full `id` string. There is no tag with the value of the `name` input. | `string` | `null` | no |
| <a name="input_namespace"></a> [namespace](#input\_namespace) | ID element. Usually an abbreviation of your organization name, e.g. 'eg' or 'cp', to help ensure generated IDs are globally unique | `string` | `null` | no |
| <a name="input_nat_elastic_ips"></a> [nat\_elastic\_ips](#input\_nat\_elastic\_ips) | Existing Elastic IPs (not EIP IDs) to attach to the NAT Gateway(s) or Instance(s) instead of creating new ones. | `list(string)` | `[]` | no |
| <a name="input_nat_gateway_enabled"></a> [nat\_gateway\_enabled](#input\_nat\_gateway\_enabled) | Set `true` to create NAT Gateways to perform IPv4 NAT and NAT64 as needed.<br>Defaults to `true` unless `nat_instance_enabled` is `true`. | `bool` | `null` | no |
| <a name="input_nat_instance_ami_id"></a> [nat\_instance\_ami\_id](#input\_nat\_instance\_ami\_id) | A list optionally containing the ID of the AMI to use for the NAT instance.<br>If the list is empty (the default), the latest official AWS NAT instance AMI<br>will be used. NOTE: The Official NAT instance AMI is being phased out and<br>does not support NAT64. Use of a NAT gateway is recommended instead. | `list(string)` | `[]` | no |
| <a name="input_nat_instance_cpu_credits_override"></a> [nat\_instance\_cpu\_credits\_override](#input\_nat\_instance\_cpu\_credits\_override) | NAT Instance credit option for CPU usage. Valid values are "standard" or "unlimited".<br>T3 and later instances are launched as unlimited by default. T2 instances are launched as standard by default. | `string` | `""` | no |
| <a name="input_nat_instance_enabled"></a> [nat\_instance\_enabled](#input\_nat\_instance\_enabled) | Set `true` to create NAT Instances to perform IPv4 NAT.<br>Defaults to `false`. | `bool` | `null` | no |
| <a name="input_nat_instance_root_block_device_encrypted"></a> [nat\_instance\_root\_block\_device\_encrypted](#input\_nat\_instance\_root\_block\_device\_encrypted) | Whether to encrypt the root block device on the created NAT instances | `bool` | `true` | no |
| <a name="input_nat_instance_type"></a> [nat\_instance\_type](#input\_nat\_instance\_type) | NAT Instance type | `string` | `"t3.micro"` | no |
| <a name="input_open_network_acl_ipv4_rule_number"></a> [open\_network\_acl\_ipv4\_rule\_number](#input\_open\_network\_acl\_ipv4\_rule\_number) | The `rule_no` assigned to the network ACL rules for IPv4 traffic generated by this module | `number` | `100` | no |
| <a name="input_open_network_acl_ipv6_rule_number"></a> [open\_network\_acl\_ipv6\_rule\_number](#input\_open\_network\_acl\_ipv6\_rule\_number) | The `rule_no` assigned to the network ACL rules for IPv6 traffic generated by this module | `number` | `111` | no |
| <a name="input_private_assign_ipv6_address_on_creation"></a> [private\_assign\_ipv6\_address\_on\_creation](#input\_private\_assign\_ipv6\_address\_on\_creation) | If `true`, network interfaces created in a private subnet will be assigned an IPv6 address | `bool` | `true` | no |
| <a name="input_private_dns64_nat64_enabled"></a> [private\_dns64\_nat64\_enabled](#input\_private\_dns64\_nat64\_enabled) | If `true` and IPv6 is enabled, DNS queries made to the Amazon-provided DNS Resolver in private subnets will return synthetic<br>IPv6 addresses for IPv4-only destinations, and these addresses will be routed to the NAT Gateway.<br>Requires `public_subnets_enabled`, `nat_gateway_enabled`, and `private_route_table_enabled` to be `true` to be fully operational.<br>Defaults to `true` unless there is no public IPv4 subnet for egress, in which case it defaults to `false`. | `bool` | `null` | no |
| <a name="input_private_label"></a> [private\_label](#input\_private\_label) | The string to use in IDs and elsewhere to identify resources for the private subnets and distinguish them from resources for the public subnets | `string` | `"private"` | no |
| <a name="input_private_open_network_acl_enabled"></a> [private\_open\_network\_acl\_enabled](#input\_private\_open\_network\_acl\_enabled) | If `true`, a single network ACL be created and it will be associated with every private subnet, and a rule (number 100)<br>will be created allowing all ingress and all egress. You can add additional rules to this network ACL<br>using the `aws_network_acl_rule` resource.<br>If `false`, you will need to manage the network ACL outside of this module. | `bool` | `true` | no |
| <a name="input_private_route_table_enabled"></a> [private\_route\_table\_enabled](#input\_private\_route\_table\_enabled) | If `true`, a network route table and default route to the NAT gateway, NAT instance, or egress-only gateway<br>will be created for each private subnet (1:1). If false, you will need to create your own route table(s) and route(s). | `bool` | `true` | no |
| <a name="input_private_subnets_additional_tags"></a> [private\_subnets\_additional\_tags](#input\_private\_subnets\_additional\_tags) | Additional tags to be added to private subnets | `map(string)` | `{}` | no |
| <a name="input_private_subnets_enabled"></a> [private\_subnets\_enabled](#input\_private\_subnets\_enabled) | If false, do not create private subnets (or NAT gateways or instances) | `bool` | `true` | no |
| <a name="input_public_assign_ipv6_address_on_creation"></a> [public\_assign\_ipv6\_address\_on\_creation](#input\_public\_assign\_ipv6\_address\_on\_creation) | If `true`, network interfaces created in a public subnet will be assigned an IPv6 address | `bool` | `true` | no |
| <a name="input_public_dns64_nat64_enabled"></a> [public\_dns64\_nat64\_enabled](#input\_public\_dns64\_nat64\_enabled) | If `true` and IPv6 is enabled, DNS queries made to the Amazon-provided DNS Resolver in public subnets will return synthetic<br>IPv6 addresses for IPv4-only destinations, and these addresses will be routed to the NAT Gateway.<br>Requires `nat_gateway_enabled` and `public_route_table_enabled` to be `true` to be fully operational. | `bool` | `false` | no |
| <a name="input_public_label"></a> [public\_label](#input\_public\_label) | The string to use in IDs and elsewhere to identify resources for the public subnets and distinguish them from resources for the private subnets | `string` | `"public"` | no |
| <a name="input_public_open_network_acl_enabled"></a> [public\_open\_network\_acl\_enabled](#input\_public\_open\_network\_acl\_enabled) | If `true`, a single network ACL be created and it will be associated with every public subnet, and a rule<br>will be created allowing all ingress and all egress. You can add additional rules to this network ACL<br>using the `aws_network_acl_rule` resource.<br>If `false`, you will need to manage the network ACL outside of this module. | `bool` | `true` | no |
| <a name="input_public_route_table_enabled"></a> [public\_route\_table\_enabled](#input\_public\_route\_table\_enabled) | If `true`, network route table(s) will be created as determined by `public_route_table_per_subnet_enabled` and<br>appropriate routes will be added to destinations this module knows about.<br>If `false`, you will need to create your own route table(s) and route(s).<br>Ignored if `public_route_table_ids` is non-empty. | `bool` | `true` | no |
| <a name="input_public_route_table_ids"></a> [public\_route\_table\_ids](#input\_public\_route\_table\_ids) | List optionally containing the ID of a single route table shared by all public subnets<br>or exactly one route table ID for each public subnet.<br>If provided, it overrides `public_route_table_per_subnet_enabled`.<br>If omitted and `public_route_table_enabled` is `true`,<br>one or more network route tables will be created for the public subnets,<br>according to the setting of `public_route_table_per_subnet_enabled`. | `list(string)` | `[]` | no |
| <a name="input_public_route_table_per_subnet_enabled"></a> [public\_route\_table\_per\_subnet\_enabled](#input\_public\_route\_table\_per\_subnet\_enabled) | If `true` (and `public_route_table_enabled` is `true`), a separate network route table will be created for and associated with each public subnet.<br>If `false` (and `public_route_table_enabled` is `true`), a single network route table will be created and it will be associated with every public subnet.<br>If not set, it will be set to the value of `public_dns64_nat64_enabled`. | `bool` | `null` | no |
| <a name="input_public_subnets_additional_tags"></a> [public\_subnets\_additional\_tags](#input\_public\_subnets\_additional\_tags) | Additional tags to be added to public subnets | `map(string)` | `{}` | no |
| <a name="input_public_subnets_enabled"></a> [public\_subnets\_enabled](#input\_public\_subnets\_enabled) | If false, do not create public subnets.<br>Since NAT gateways and instances must be created in public subnets, these will also not be created when `false`. | `bool` | `true` | no |
| <a name="input_regex_replace_chars"></a> [regex\_replace\_chars](#input\_regex\_replace\_chars) | Terraform regular expression (regex) string.<br>Characters matching the regex will be removed from the ID elements.<br>If not set, `"/[^a-zA-Z0-9-]/"` is used to remove all characters other than hyphens, letters and digits. | `string` | `null` | no |
| <a name="input_root_block_device_encrypted"></a> [root\_block\_device\_encrypted](#input\_root\_block\_device\_encrypted) | DEPRECATED: use `nat_instance_root_block_device_encrypted` instead.<br>Whether to encrypt the root block device on the created NAT instances | `bool` | `null` | no |
| <a name="input_route_create_timeout"></a> [route\_create\_timeout](#input\_route\_create\_timeout) | Time to wait for a network routing table entry to be created, specified as a Go Duration, e.g. `2m` | `string` | `"5m"` | no |
| <a name="input_route_delete_timeout"></a> [route\_delete\_timeout](#input\_route\_delete\_timeout) | Time to wait for a network routing table entry to be deleted, specified as a Go Duration, e.g. `2m` | `string` | `"10m"` | no |
| <a name="input_stage"></a> [stage](#input\_stage) | ID element. Usually used to indicate role, e.g. 'prod', 'staging', 'source', 'build', 'test', 'deploy', 'release' | `string` | `null` | no |
| <a name="input_subnet_create_timeout"></a> [subnet\_create\_timeout](#input\_subnet\_create\_timeout) | Time to wait for a subnet to be created, specified as a Go Duration, e.g. `2m` | `string` | `"10m"` | no |
| <a name="input_subnet_delete_timeout"></a> [subnet\_delete\_timeout](#input\_subnet\_delete\_timeout) | Time to wait for a subnet to be deleted, specified as a Go Duration, e.g. `5m` | `string` | `"20m"` | no |
| <a name="input_subnet_type_tag_key"></a> [subnet\_type\_tag\_key](#input\_subnet\_type\_tag\_key) | DEPRECATED: Use `public_subnets_additional_tags` and `private_subnets_additional_tags` instead<br>Key for subnet type tag to provide information about the type of subnets, e.g. `cpco.io/subnet/type: private` or `cpco.io/subnet/type: public` | `string` | `null` | no |
| <a name="input_subnet_type_tag_value_format"></a> [subnet\_type\_tag\_value\_format](#input\_subnet\_type\_tag\_value\_format) | DEPRECATED: Use `public_subnets_additional_tags` and `private_subnets_additional_tags` instead.<br>The value of the `subnet_type_tag_key` will be set to `format(var.subnet_type_tag_value_format, <type>)`<br>where `<type>` is either `public` or `private`. | `string` | `"%s"` | no |
| <a name="input_subnets_per_az_count"></a> [subnets\_per\_az\_count](#input\_subnets\_per\_az\_count) | The number of subnet of each type (public or private) to provision per Availability Zone. | `number` | `1` | no |
| <a name="input_subnets_per_az_names"></a> [subnets\_per\_az\_names](#input\_subnets\_per\_az\_names) | The subnet names of each type (public or private) to provision per Availability Zone.<br>This variable is optional.<br>If a list of names is provided, the list items will be used as keys in the outputs `named_private_subnets_map`, `named_public_subnets_map`,<br>`named_private_route_table_ids_map` and `named_public_route_table_ids_map` | `list(string)` | <pre>[<br>  "common"<br>]</pre> | no |
| <a name="input_tags"></a> [tags](#input\_tags) | Additional tags (e.g. `{'BusinessUnit': 'XYZ'}`).<br>Neither the tag keys nor the tag values will be modified by this module. | `map(string)` | `{}` | no |
| <a name="input_tenant"></a> [tenant](#input\_tenant) | ID element \_(Rarely used, not included by default)\_. A customer identifier, indicating who this instance of a resource is for | `string` | `null` | no |
| <a name="input_vpc_id"></a> [vpc\_id](#input\_vpc\_id) | VPC ID where subnets will be created (e.g. `vpc-aceb2723`) | `string` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_availability_zone_ids"></a> [availability\_zone\_ids](#output\_availability\_zone\_ids) | List of Availability Zones IDs where subnets were created, when available |
| <a name="output_availability_zones"></a> [availability\_zones](#output\_availability\_zones) | List of Availability Zones where subnets were created |
| <a name="output_az_private_route_table_ids_map"></a> [az\_private\_route\_table\_ids\_map](#output\_az\_private\_route\_table\_ids\_map) | Map of AZ names to list of private route table IDs in the AZs |
| <a name="output_az_private_subnets_map"></a> [az\_private\_subnets\_map](#output\_az\_private\_subnets\_map) | Map of AZ names to list of private subnet IDs in the AZs |
| <a name="output_az_public_route_table_ids_map"></a> [az\_public\_route\_table\_ids\_map](#output\_az\_public\_route\_table\_ids\_map) | Map of AZ names to list of public route table IDs in the AZs |
| <a name="output_az_public_subnets_map"></a> [az\_public\_subnets\_map](#output\_az\_public\_subnets\_map) | Map of AZ names to list of public subnet IDs in the AZs |
| <a name="output_named_private_route_table_ids_map"></a> [named\_private\_route\_table\_ids\_map](#output\_named\_private\_route\_table\_ids\_map) | Map of subnet names (specified in `subnets_per_az_names` variable) to lists of private route table IDs |
| <a name="output_named_private_subnets_map"></a> [named\_private\_subnets\_map](#output\_named\_private\_subnets\_map) | Map of subnet names (specified in `subnets_per_az_names` variable) to lists of private subnet IDs |
| <a name="output_named_private_subnets_stats_map"></a> [named\_private\_subnets\_stats\_map](#output\_named\_private\_subnets\_stats\_map) | Map of subnet names (specified in `subnets_per_az_names` variable) to lists of objects with each object having three items: AZ, private subnet ID, private route table ID |
| <a name="output_named_public_route_table_ids_map"></a> [named\_public\_route\_table\_ids\_map](#output\_named\_public\_route\_table\_ids\_map) | Map of subnet names (specified in `subnets_per_az_names` variable) to lists of public route table IDs |
| <a name="output_named_public_subnets_map"></a> [named\_public\_subnets\_map](#output\_named\_public\_subnets\_map) | Map of subnet names (specified in `subnets_per_az_names` variable) to lists of public subnet IDs |
| <a name="output_named_public_subnets_stats_map"></a> [named\_public\_subnets\_stats\_map](#output\_named\_public\_subnets\_stats\_map) | Map of subnet names (specified in `subnets_per_az_names` variable) to lists of objects with each object having three items: AZ, public subnet ID, public route table ID |
| <a name="output_nat_eip_allocation_ids"></a> [nat\_eip\_allocation\_ids](#output\_nat\_eip\_allocation\_ids) | Elastic IP allocations in use by NAT |
| <a name="output_nat_gateway_ids"></a> [nat\_gateway\_ids](#output\_nat\_gateway\_ids) | IDs of the NAT Gateways created |
| <a name="output_nat_gateway_public_ips"></a> [nat\_gateway\_public\_ips](#output\_nat\_gateway\_public\_ips) | DEPRECATED: use `nat_ips` instead. Public IPv4 IP addresses in use by NAT. |
| <a name="output_nat_instance_ami_id"></a> [nat\_instance\_ami\_id](#output\_nat\_instance\_ami\_id) | ID of AMI used by NAT instance |
| <a name="output_nat_instance_ids"></a> [nat\_instance\_ids](#output\_nat\_instance\_ids) | IDs of the NAT Instances created |
| <a name="output_nat_ips"></a> [nat\_ips](#output\_nat\_ips) | Elastic IP Addresses in use by NAT |
| <a name="output_private_network_acl_id"></a> [private\_network\_acl\_id](#output\_private\_network\_acl\_id) | ID of the Network ACL created for private subnets |
| <a name="output_private_route_table_ids"></a> [private\_route\_table\_ids](#output\_private\_route\_table\_ids) | IDs of the created private route tables |
| <a name="output_private_subnet_cidrs"></a> [private\_subnet\_cidrs](#output\_private\_subnet\_cidrs) | IPv4 CIDR blocks of the created private subnets |
| <a name="output_private_subnet_ids"></a> [private\_subnet\_ids](#output\_private\_subnet\_ids) | IDs of the created private subnets |
| <a name="output_private_subnet_ipv6_cidrs"></a> [private\_subnet\_ipv6\_cidrs](#output\_private\_subnet\_ipv6\_cidrs) | IPv6 CIDR blocks of the created private subnets |
| <a name="output_public_network_acl_id"></a> [public\_network\_acl\_id](#output\_public\_network\_acl\_id) | ID of the Network ACL created for public subnets |
| <a name="output_public_route_table_ids"></a> [public\_route\_table\_ids](#output\_public\_route\_table\_ids) | IDs of the created public route tables |
| <a name="output_public_subnet_cidrs"></a> [public\_subnet\_cidrs](#output\_public\_subnet\_cidrs) | IPv4 CIDR blocks of the created public subnets |
| <a name="output_public_subnet_ids"></a> [public\_subnet\_ids](#output\_public\_subnet\_ids) | IDs of the created public subnets |
| <a name="output_public_subnet_ipv6_cidrs"></a> [public\_subnet\_ipv6\_cidrs](#output\_public\_subnet\_ipv6\_cidrs) | IPv6 CIDR blocks of the created public subnets |
<!-- markdownlint-restore -->



## Share the Love

Like this project? Please give it a ★ on [our GitHub](https://github.com/cloudposse/terraform-aws-dynamic-subnets)! (it helps us **a lot**)

Are you using this project or any of our other projects? Consider [leaving a testimonial][testimonial]. =)



## Related Projects

Check out these related projects.

- [terraform-aws-vpc](https://github.com/cloudposse/terraform-aws-vpc) - Terraform Module that defines a VPC with public/private subnets across multiple AZs with Internet Gateways
- [terraform-aws-vpc-peering](https://github.com/cloudposse/terraform-aws-vpc-peering) - Terraform module to create a peering connection between two VPCs
- [terraform-aws-kops-vpc-peering](https://github.com/cloudposse/terraform-aws-kops-vpc-peering) - Terraform module to create a peering connection between a backing services VPC and a VPC created by Kops
- [terraform-aws-named-subnets](https://github.com/cloudposse/terraform-aws-named-subnets) - Terraform module for named subnets provisioning.

## Help

**Got a question?** We got answers.

File a GitHub [issue](https://github.com/cloudposse/terraform-aws-dynamic-subnets/issues), send us an [email][email] or join our [Slack Community][slack].

[![README Commercial Support][readme_commercial_support_img]][readme_commercial_support_link]

## DevOps Accelerator for Startups


We are a [**DevOps Accelerator**][commercial_support]. We'll help you build your cloud infrastructure from the ground up so you can own it. Then we'll show you how to operate it and stick around for as long as you need us.

[![Learn More](https://img.shields.io/badge/learn%20more-success.svg?style=for-the-badge)][commercial_support]

Work directly with our team of DevOps experts via email, slack, and video conferencing.

We deliver 10x the value for a fraction of the cost of a full-time engineer. Our track record is not even funny. If you want things done right and you need it done FAST, then we're your best bet.

- **Reference Architecture.** You'll get everything you need from the ground up built using 100% infrastructure as code.
- **Release Engineering.** You'll have end-to-end CI/CD with unlimited staging environments.
- **Site Reliability Engineering.** You'll have total visibility into your apps and microservices.
- **Security Baseline.** You'll have built-in governance with accountability and audit logs for all changes.
- **GitOps.** You'll be able to operate your infrastructure via Pull Requests.
- **Training.** You'll receive hands-on training so your team can operate what we build.
- **Questions.** You'll have a direct line of communication between our teams via a Shared Slack channel.
- **Troubleshooting.** You'll get help to triage when things aren't working.
- **Code Reviews.** You'll receive constructive feedback on Pull Requests.
- **Bug Fixes.** We'll rapidly work with you to fix any bugs in our projects.

## Slack Community

Join our [Open Source Community][slack] on Slack. It's **FREE** for everyone! Our "SweetOps" community is where you get to talk with others who share a similar vision for how to rollout and manage infrastructure. This is the best place to talk shop, ask questions, solicit feedback, and work together as a community to build totally *sweet* infrastructure.

## Discourse Forums

Participate in our [Discourse Forums][discourse]. Here you'll find answers to commonly asked questions. Most questions will be related to the enormous number of projects we support on our GitHub. Come here to collaborate on answers, find solutions, and get ideas about the products and services we value. It only takes a minute to get started! Just sign in with SSO using your GitHub account.

## Newsletter

Sign up for [our newsletter][newsletter] that covers everything on our technology radar.  Receive updates on what we're up to on GitHub as well as awesome new projects we discover.

## Office Hours

[Join us every Wednesday via Zoom][office_hours] for our weekly "Lunch & Learn" sessions. It's **FREE** for everyone!

[![zoom](https://img.cloudposse.com/fit-in/200x200/https://cloudposse.com/wp-content/uploads/2019/08/Powered-by-Zoom.png")][office_hours]

## Contributing

### Bug Reports & Feature Requests

Please use the [issue tracker](https://github.com/cloudposse/terraform-aws-dynamic-subnets/issues) to report any bugs or file feature requests.

### Developing

If you are interested in being a contributor and want to get involved in developing this project or [help out](https://cpco.io/help-out) with our other projects, we would love to hear from you! Shoot us an [email][email].

In general, PRs are welcome. We follow the typical "fork-and-pull" Git workflow.

 1. **Fork** the repo on GitHub
 2. **Clone** the project to your own machine
 3. **Commit** changes to your own branch
 4. **Push** your work back up to your fork
 5. Submit a **Pull Request** so that we can review your changes

**NOTE:** Be sure to merge the latest changes from "upstream" before making a pull request!


## Copyright

Copyright © 2017-2023 [Cloud Posse, LLC](https://cpco.io/copyright)



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









## Trademarks

All other trademarks referenced herein are the property of their respective owners.

## About

This project is maintained and funded by [Cloud Posse, LLC][website]. Like it? Please let us know by [leaving a testimonial][testimonial]!

[![Cloud Posse][logo]][website]

We're a [DevOps Professional Services][hire] company based in Los Angeles, CA. We ❤️  [Open Source Software][we_love_open_source].

We offer [paid support][commercial_support] on all of our projects.

Check out [our other projects][github], [follow us on twitter][twitter], [apply for a job][jobs], or [hire us][hire] to help with your cloud strategy and implementation.



### Contributors

<!-- markdownlint-disable -->
|  [![Erik Osterman][osterman_avatar]][osterman_homepage]<br/>[Erik Osterman][osterman_homepage] | [![Andriy Knysh][aknysh_avatar]][aknysh_homepage]<br/>[Andriy Knysh][aknysh_homepage] | [![Nuru][Nuru_avatar]][Nuru_homepage]<br/>[Nuru][Nuru_homepage] | [![Sergey Vasilyev][s2504s_avatar]][s2504s_homepage]<br/>[Sergey Vasilyev][s2504s_homepage] | [![Vladimir][SweetOps_avatar]][SweetOps_homepage]<br/>[Vladimir][SweetOps_homepage] | [![Konstantin B][comeanother_avatar]][comeanother_homepage]<br/>[Konstantin B][comeanother_homepage] | [![dcowan-vestmark][dcowan-vestmark_avatar]][dcowan-vestmark_homepage]<br/>[dcowan-vestmark][dcowan-vestmark_homepage] | [![Ivan Pinatti][ivan-pinatti_avatar]][ivan-pinatti_homepage]<br/>[Ivan Pinatti][ivan-pinatti_homepage] | [![Oscar Sullivan][osulli_avatar]][osulli_homepage]<br/>[Oscar Sullivan][osulli_homepage] | [![Joe Niland][joe-niland_avatar]][joe-niland_homepage]<br/>[Joe Niland][joe-niland_homepage] |
|---|---|---|---|---|---|---|---|---|---|
<!-- markdownlint-restore -->

  [osterman_homepage]: https://github.com/osterman
  [osterman_avatar]: https://img.cloudposse.com/150x150/https://github.com/osterman.png
  [aknysh_homepage]: https://github.com/aknysh
  [aknysh_avatar]: https://img.cloudposse.com/150x150/https://github.com/aknysh.png
  [Nuru_homepage]: https://github.com/Nuru
  [Nuru_avatar]: https://img.cloudposse.com/150x150/https://github.com/Nuru.png
  [s2504s_homepage]: https://github.com/s2504s
  [s2504s_avatar]: https://img.cloudposse.com/150x150/https://github.com/s2504s.png
  [SweetOps_homepage]: https://github.com/SweetOps
  [SweetOps_avatar]: https://img.cloudposse.com/150x150/https://github.com/SweetOps.png
  [comeanother_homepage]: https://github.com/comeanother
  [comeanother_avatar]: https://img.cloudposse.com/150x150/https://github.com/comeanother.png
  [dcowan-vestmark_homepage]: https://github.com/dcowan-vestmark
  [dcowan-vestmark_avatar]: https://img.cloudposse.com/150x150/https://github.com/dcowan-vestmark.png
  [ivan-pinatti_homepage]: https://github.com/ivan-pinatti
  [ivan-pinatti_avatar]: https://img.cloudposse.com/150x150/https://github.com/ivan-pinatti.png
  [osulli_homepage]: https://github.com/osulli
  [osulli_avatar]: https://img.cloudposse.com/150x150/https://github.com/osulli.png
  [joe-niland_homepage]: https://github.com/joe-niland
  [joe-niland_avatar]: https://img.cloudposse.com/150x150/https://github.com/joe-niland.png

[![README Footer][readme_footer_img]][readme_footer_link]
[![Beacon][beacon]][website]
<!-- markdownlint-disable -->
  [logo]: https://cloudposse.com/logo-300x69.svg
  [docs]: https://cpco.io/docs?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=docs
  [website]: https://cpco.io/homepage?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=website
  [github]: https://cpco.io/github?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=github
  [jobs]: https://cpco.io/jobs?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=jobs
  [hire]: https://cpco.io/hire?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=hire
  [slack]: https://cpco.io/slack?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=slack
  [linkedin]: https://cpco.io/linkedin?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=linkedin
  [twitter]: https://cpco.io/twitter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=twitter
  [testimonial]: https://cpco.io/leave-testimonial?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=testimonial
  [office_hours]: https://cloudposse.com/office-hours?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=office_hours
  [newsletter]: https://cpco.io/newsletter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=newsletter
  [discourse]: https://ask.sweetops.com/?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=discourse
  [email]: https://cpco.io/email?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=email
  [commercial_support]: https://cpco.io/commercial-support?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=commercial_support
  [we_love_open_source]: https://cpco.io/we-love-open-source?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=we_love_open_source
  [terraform_modules]: https://cpco.io/terraform-modules?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=terraform_modules
  [readme_header_img]: https://cloudposse.com/readme/header/img
  [readme_header_link]: https://cloudposse.com/readme/header/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=readme_header_link
  [readme_footer_img]: https://cloudposse.com/readme/footer/img
  [readme_footer_link]: https://cloudposse.com/readme/footer/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=readme_footer_link
  [readme_commercial_support_img]: https://cloudposse.com/readme/commercial-support/img
  [readme_commercial_support_link]: https://cloudposse.com/readme/commercial-support/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-dynamic-subnets&utm_content=readme_commercial_support_link
  [share_twitter]: https://twitter.com/intent/tweet/?text=terraform-aws-dynamic-subnets&url=https://github.com/cloudposse/terraform-aws-dynamic-subnets
  [share_linkedin]: https://www.linkedin.com/shareArticle?mini=true&title=terraform-aws-dynamic-subnets&url=https://github.com/cloudposse/terraform-aws-dynamic-subnets
  [share_reddit]: https://reddit.com/submit/?url=https://github.com/cloudposse/terraform-aws-dynamic-subnets
  [share_facebook]: https://facebook.com/sharer/sharer.php?u=https://github.com/cloudposse/terraform-aws-dynamic-subnets
  [share_googleplus]: https://plus.google.com/share?url=https://github.com/cloudposse/terraform-aws-dynamic-subnets
  [share_email]: mailto:?subject=terraform-aws-dynamic-subnets&body=https://github.com/cloudposse/terraform-aws-dynamic-subnets
  [beacon]: https://ga-beacon.cloudposse.com/UA-76589703-4/cloudposse/terraform-aws-dynamic-subnets?pixel&cs=github&cm=readme&an=terraform-aws-dynamic-subnets
<!-- markdownlint-restore -->
