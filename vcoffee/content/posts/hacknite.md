---
title: "Hacknite"
date: 2021-02-23T15:04:17+01:00
draft: true
---


Instructions and an example of how to build a custom Terraform provider in a Docker. Comes with instructions to install a sample web application that Terraform will interact with.


The walk-through process below will let you compile a custom Terraform provider and interact with a sample web application. 

## Assumptions and Format

The format used below assumes your provider is leveraging the following syntax.

Obviously if you want to use your build your own custom Terraform provider, you can replace "vmware.com", "edu" and "ctoa" as per your preference.

For more details, check out the details on the HashiCorp [website](https://www.terraform.io/docs/language/providers/requirements.html).

```hcl
terraform {
  required_providers {
    ctoa = {
      source  = "vmware.com/edu/ctoa"
    }
  }
}
```

## Requirements:

Docker installed on your machine

### 0. Start the container ###

```bash
docker run -p 5901:5901 -p 6901:6901 -p 80:80 --user 0 adeleporte/terraform-hacknite:latest
```
and

```bash
http://127.0.0.1:6901/?password=vncpassword in a browser to get a full desktop with all requirements set (Visual Studio, GO, Terraform, Git)
```

![docker](docker.png)

### 1. Cloning the repo: ###
From a terminal:

Clone the following [repo](https://github.com/adeleporte/ctoa-hacknite.git) with the following command:  

```bash
git clone https://github.com/adeleporte/ctoa-hacknite.git
```


Navigate to the Terraform folder:  

```bash
cd ctoa-hacknite/terraform-provider-ctoa
```

Create a folder where the compiled provider will be moved to.

- If your physical machine is a MAC:
```bash
mkdir -p ~/.terraform.d/plugins/vmware.com/edu/ctoa/0.1/darwin_amd64
```

- If your physical machine is a Linux:
```bash
mkdir -p ~/.terraform.d/plugins/vmware.com/edu/ctoa/0.1/linux_amd64
```

### 2. Compiling the provider and moving to the correct folder: ###

Compile the provider:  

```bash
go build -o terraform-provider-ctoa
```

Move the provider to the correct location:  

- If your physical machine is a MAC:
```bash
mv terraform-provider-ctoa ~/.terraform.d/plugins/vmware.com/edu/ctoa/0.1/darwin_amd64/terraform-provider-ctoa
```

- If your physical machine is a Linux:
```bash
mv terraform-provider-ctoa ~/.terraform.d/plugins/vmware.com/edu/ctoa/0.1/linux_amd64/terraform-provider-ctoa
```

### 3. Start the webserver ###

To start the webserver, go to the ctoa-web folder:  

```bash
cd ..\frontend\dist\ctoa-web\
```

And start the web server:

```bash
.\cto-api-linux
```

![web server](web-server.png)
  
Don't close the windows above.
 
Go to your browser on 127.0.0.1 and you should see a basic webserver.

![web_server_pre_apply](clarity-web-server-empty.png)

### 4. Deploy and manager resources with your custom Terraform provider ###

From Visual Studio Code, navigate back to the Terraform folder:  

```bash
cd ctoa-hacknite/terraform-provider-ctoa`
```

Assuming you're in `terraform-provider-ctoa` and in the same folder as the `main.tf` file, the initialization should work:  
```bash
terraform init
```

Update the `main.tf` file with more resources. With this Terraform provider, every 'resource' you will create is a user, with a first name and a last name. For example:

```hcl
resource "ctoa_people" "adeleporte" {
  first_name = "Antoine"
  last_name = "Deleporte"
}

resource "ctoa_people" "nvibert" {
  first_name = "Nico"
  last_name = "Vibert"
}
```

In the `main.tf` file, we refer to the host as the webserver (`127.0.0.1` is the client itself). That's the webserver currently running.
 
In your Terraform terminal, once the main.tf file is updated with the resources you are creating, do a:
 
```bash
terraform plan
```
 
And a:
 
```bash
terraform apply
```
   
And you should see new entries added to the table on the webserver.

![web_server_apply](clarity-web-server-done.png)

A `terraform destroy` will remove all entries from the table.

Try to modify the resources by changing the first name or the second name of the resources. When you run a `terraform plan`, you can see that the resource is not deleted and re-created by the change, but updated-in-place.

Well done for building your own custom Terraform provider!