# Diego CI

This repository contains tools to deploy Diego to AWS environments as well as our concourse scripts.

## Deploying CF and Diego to AWS

### Local System Requirements

* [Go 1.4.3](https://github.com/kr/godep.git)
* [godep](https://github.com/tools/godep)
```
go get -u github.com/kr/godep
```
* [boosh](https://github.com/vito/boosh)
```
godep get github.com/vito/boosh
```
* [spiff](https://github.com/cloudfoundry-incubator/spiff)
```
godep get github.com/cloudfoundry-incubator/spiff
```
* The [aws cli](https://aws.amazon.com/cli/) requires python and pip to be installed
on your host machine.
```
pip install awscli
```
* [jq version 1.5+](https://stedolan.github.io/jq/)
* [Ruby 2+](https://www.ruby-lang.org/en/documentation/installation/) 
* [Bosh](http://bosh.io/) cli 
```
gem install bosh_cli
```
* [Bosh init](https://bosh.io/docs/install-bosh-init.html)

### AWS Requirements

1. Create a local directory which will be used to store your deployment-specific credentials. From here on, we will refer to
   this directory as `DEPLOYMENT_DIR`.

1. Create an AWS keypair for yout bosh director
  1.  From your AWS EC2 page click on the `Key Pairs` tab
  2.  Select the `create keypair` button at the top of the page
  3.  When prompted for the key name, enter `bosh`
  4.  Move the downloaded `bosh.pem` key to `DEPLOYMENT_DIR/keypair/` and rename the key to `id_rsa_bosh` 
  
1. Create Route 53 Hosted Zone
  1.  From the aws console homepage click on `Route 53`
  2.  Select `hosted zones` from the left sidebar
  3.  Click the `Create Hosted Zone` button
  4.  Fill in the domain name for your cloud foundry deployment
  
  By default, the domain name for your hosted zone will be the root domain of all apps deployed to your cloud foundry instance.
 
  eg:
   ```
   domain = foo.bar.com
   app name = `hello-world`. This will create a default route of hello-world.domain

   http://hello-world.foo.bar.com will be the root url address of your application
   ```

### System Setup

Your DEPLOYMENT_DIR needs to have the following the following format:
```
DEPLOYMENT_DIR
|-(bootstrap_environment)
|-keypair
| |-(id_rsa_bosh)
|-certs
| |-(elb-cfrouter.key)
| |-(elb-cfrouter.pem)
|-stubs
| |-(domain.yml)
| |-infrastructure
| | |-(availablity_zones.yml)  # available aws zones change whenever the heck they feel like it
| |-bosh-init
|   |-(releases.yml)
|   |-(users.yml)
|   |-(stemcell.yml)
```

#### bootstrap_environment

This script should export your aws default region and access/secret keys as environment variables.
The `AWS_ACCESS_KEY_ID` key must match your aws IAM user's access key id and the `AWS_SECRET_ACCESS_KEY`
is the private key generated during the IAM user creation.
 
eg:
```
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=xxxxxxxxxxxxxxxxxxx
export AWS_SECRET_ACCESS_KEY='xxxxxxxxxxxxxxxxxxxxxx'
```

#### keypair/id_rsa_bosh

This is the private key pair we generated for our BOSH director when we [created an AWS keypair](#aws-requirements).

#### certs/elb-cfrouter.key && certs/elb-cfrouter.pem
An SSL certificate for the domain where Cloud Foundry will be accessible is required. If you do not provide a certificate,
you can generate a self signed cert following the commands below:

```
openssl genrsa -out elb-cfrouter.key 2048
openssl req -new -key elb-cfrouter.key -out elb-cfrouter.csr
openssl x509 -req -in elb-cfrouter.csr -signkey elb-cfrouter.key -out elb-cfrouter.pem
```

#### stubs/domain.yml

The `domain.yml` should be assigned to the domain we generated when we [created a route 53 hosted zone](#aws-requirements).

eg:
```
---
domain: <your-domain.com>
```

#### stubs/infrastructure/availability_zones.yml

This yaml file defines the 3 zones that will host your Cloud Foundry Deployment.

eg:
```
meta:
  availability_zones:
    - us-east-1a
    - us-east-1c
    - us-east-1d
``` 

Note: These zones could become restricted by AWS. If at some point during the `deploy_aws_cli` script and you see an error
with the following message: "Value (us-east-1b) for parameter availabilityZone is invalid. Subnets can currently
only be created in the following availability zones: us-east-1d, us-east-1b, us-east-1a, us-east-1e," you will need
to update this file with acceptable values.

#### stubs/bosh-init/releases.yml

To deploy the bosh director, bosh-init's `releases.yml` must specify `bosh` and `bosh-aws-cpi` releases by url and sha1.

eg:
```
releases:
  - name: bosh
    url: https://bosh.io/d/github.com/cloudfoundry/bosh?v=210
    sha1: 0ff01bfe8ead91ff2c4cfe5309a1c60b344aeb09
  - name: bosh-aws-cpi
    url: https://bosh.io/d/github.com/cloudfoundry-incubator/bosh-aws-cpi-release?v=31
    sha1: bde15dfb3e4f1b9e9693c810fa539858db2bc298
```
 
Releases for `bosh` can be found [here](http://bosh.io/releases/github.com/cloudfoundry/bosh?all=1).
Releases for `bosh-aws-cpi` can be found [here](http://bosh.io/releases/github.com/cloudfoundry-incubator/bosh-aws-cpi-release?all=1).   

#### stubs/bosh-init/users.yml

This file defines the admin users for your bosh director.

eg:
```
bosh-init:
  users:
    - {name: admin, password: YOUR_PASSWORD}
```

#### stubs/bosh-init/stemcell.yml

Stemcell.yml defines which stemcell to use on the bosh director. Stemcells can be found
[here](http://bosh.io/stemcells/bosh-aws-xen-ubuntu-trusty-go_agent), and must be specified by their url and sha1.

eg:
```
stemcell:
  url: https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3091
  sha1: 21ce6eb039179bb5b1706adfea4c161ea20dea1f 
```

# Creating the CF AWS environment

To create the AWS environment and a couple of VMs essential to the Cloud Foundry infrastructure,
you will need to run `./deploy_aws_environment create DEPLOYMENT_DIR` from this repository.

This script can be asked to `create`, `update`, or `skip` our Cloud Foundry AWS environment. The first time we run this, we will specify `create` to spin up an AWS Cloud Formation Stack based off of the stubs we filled out above. Subsequently, we will only want to provide `update` if we change stubs under DEPLOYMENT_DIR/stubs/infrastructure or there was an update to this repository. Otherwise we will want to run the script with `skip` so that no unnecessary changes are applied to the AWS environment.

The second parameter is your DEPLOYMENT_DIR and must be structured as defined above. During our deployment process
we will generate a few additional stubs that include the line "GENERATED: NO TOUCHING".

Outputs:
```
deployment_dir
|-stubs
| |-(directort-uuid.yml) # the bosh director's unique id
| |-(aws-resources.yml)  # general metadata about our cloudformation deployment
| |-cf
| | |-(aws.yml) # networks, zones, s3 buckets for our CF deployment
| |-infrastructure
|   |-(certificates.yml) # for our aws-provided elb
|   |-(cloudformation.json) # aws' deployed cloudformation.json
|-deployments
| |-bosh-init
|   |-(bosh-init.yml) # bosh director deployment
```

# Deploying CF

To deploy Cloud Foundry against our newly created AWS environment we need to generate a manifest for our deployment.
The instructions for full manifest generation can be found [here](http://docs.cloudfoundry.org/deploying/common/create_a_manifest.html),
but the part we are most concerned with is the section "Create a Deployment Manifest Stub" as the `./deploy_aws_environment` script generated `director-uuid.yml` for us.

We need to create a stub which looks like the one from the Cloud Foundry Documentation
[here](http://docs.cloudfoundry.org/deploying/aws/cf-stub.html) with a few minor tweaks. Not to fret though, creating our AWS environment above should have generated a `DEPLOYMENT_DIR/stubs/cf/aws.yml`
that can be used as a starting point for generating a diego-compatible stub. This stub has some additional properties that
can be used to deploy CF across 3 zones instead of the 2 shown in the Cloud Foundry Docs. Don't worry about that
extra information, just follow along with the Cloud Foundry [editing instructions](http://docs.cloudfoundry.org/deploying/aws/cf-stub.html#editing) and fill in any properties that aren't already specified in your generated cf/aws.yml stub.

Default manifest generation won't create some VMs and properties that Diego depends on and leave around some unnecessary legacy VMs and properties that Diego doesn't need. To correct this we can use the provided cf stub under `./stubs/cf/diego.yml` when generating our Cloud Foundry manifest.

After following the instructions to generate a `cf/aws.yml` stub and downloading the cf-release directory, we can run
the following command inside our diego-ci repository to generate our Cloud Foundry manifest:
```
spiff merge CF-RELEASE_DIRECTORY/scripts/generate_deployment_manifest aws \
DEPLOYMENT_DIR/stubs/director_uuid.yml \
DEPLOYMENT_DIR/stubs/cf/aws.yml ./stubs/cf/diego.yml \
> DEPLOYMENT_DIR/deployments/cf.yml
```

From here, all we need to do is [deploy](http://docs.cloudfoundry.org/deploying/common/deploy.html). Just remember to use our generated deployment manifest with the command `bosh deployment DEPLOYMENT_DIR/deployment/cf.yml`.

# Deploying Diego

To deploy Diego follow the instructions on the [Diego Release](https://github.com/cloudfoundry-incubator/diego-release) page.
