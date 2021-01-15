---
title: "VeloCloud Terraform Provider published on Hashicorp registry!"
date: 2021-01-15T15:01:14+01:00
draft: false
featured: true
tags: [
    "velocloud",
    "terraform",
    "code"
]
---

My Terraform provider for VeloCloud is now available on the official Terraform registry. Checkout [https://registry.terraform.io/providers/adeleporte/velocloud/latest](https://registry.terraform.io/providers/adeleporte/velocloud/latest)

For me, it was the opportunity to learn more about:

* signing some code
* managing the GitHub release process
* using go-releaser
* documenting the code with Terraform syntax

The provider is published as a community provider, so with no official VMware support.

By the way, as of today, it enables:

* the creation/update/deletion of Edges
* the creation/update/deletion of Address Groups
* the creation/update/deletion of Port Groups
* the management of Business Policies
* the management of Firewall Rules
* the management of Interface Settings within the Device Settings module

Some datasources are also available:

* Existing Port Group
* Existing Address Group
* Existing Profile
* Existing Edge
* Map an application

Future release will include Operator-level operations, like:

* the creation/update/deletion of Enterprises
* the creation/update/deletion of Gateways

and day-0 operations:

* VCO installation

Start trying the provider with:

```hcl
terraform {
  required_providers {
    velocloud   = {
      source    = "adeleporte/velocloud"
      version   = "0.2.9"
    }
  }
}

 
provider "velocloud" {
  # Configuration options
}
```

And learn how to use it with https://registry.terraform.io/providers/adeleporte/velocloud/latest/docs

