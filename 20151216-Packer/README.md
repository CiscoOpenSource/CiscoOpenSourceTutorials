# Using Packer with OpenStack
### December 16, 2015
### Vallard Benincosa [@vallard](http://twitter.com/vallard)

[Packer](http://packer.io) is a great tool because it allows you to easily create and distribute
the same image on every cloud platform.  

Packer helps with two big problems:
* Black Box Golden Images.
* Getting the same image to all your cloud platforms. 

#### Avoiding Black Boxes

The first issue 'Black Box Golden Images' is the fact that you may have an image you use for running
your production environment.  This image may have been baked long ago and nobody knows how to reproduce
exactly.  The answer to this problem is source code revision control.  Packer allows you to create a 
script to store and execute this image.  Any time you change it, you update the packer file that is 
stored in a repository like Git. 

Many people (myself included) have stayed away from golden images and instead build images on the fly. 
With AWS and OpenStack we're blessed to have cloud-init in our life.  The problem with [cloud-init](https://cloudinit.readthedocs.org/en/latest/) is that
depending on how complicated your image is, it can take a long time to bring it up. When we use a 
CI/CD pipeline approach, we want bringing new instance environments to take close to 0 time to speed our
testing cycles.  

Since Packer has the benefits of not being a black box and of being faster than using ```cloud-init```
it is a perfect tool to use in our deployment pipeline.  

#### One Image on every cloud

The second issue Packer resolves is the laborious process of getting the same image to all the different clouds
you may be using.  Packer has support for:

* OpenStack
* AWS
* Digital Ocean
* VMware

And several more. If you were operating on all of those clouds you could easily bake one native image that looked the
same on all of them.  This is the beauty of Packer and what makes it so useful. 

Of course you could use a tool like Ansible or other configuration management tools (and many do) the problem is it 
takes a few more steps than what you can do with Packer.  Instead, you can have Packer call those configuration management
tools during the build process and have the best of both worlds. 

### Packer and OpenStack

The [Documentation](https://www.packer.io/docs/builders/openstack.html) on Packer's OpenStack capabilities is 
pretty good.  Currently there are a few gotchas I have run into I thought I would point out.  

#### A basic Example

Running Jenkins Slaves is one of the primary reasons I use Packer.  I create a basic file named ```coreos-jenkins.json```
The contents of the file look as follows: 
```
{
  "_floating_ip": "xx.xx.xx.xx",
  "builders" : [{
    "floating_ip_pool": "nova",
    "type": "openstack",
    "ssh_username": "core", 
    "image_name": "coreos-jenkinslave-{{ timestamp }}",
    "source_image" : "48342e8e-e24e-4732-8a56-66d6d29e1931",
    "flavor": "m1.small",
  }
],
  "provisioners": [{
    "type": "shell",
    "script" : "jenkins_setup.sh"
  }]  
}
```

The only real requirements you need to have are: 
* The ```image_name```: This is the name you want the image to be when you are done. You'll notice in this example I use the ```{{ timestamp }}``` built in variable to timestamp my image name. 
* The ```source_image```: This is the existing image you have that you want to start from.  Using ```nova image-list``` will give you the images to choose from.  I'm using a CoreOS image.  
* The ```flavor```: For my tests using the m1.small works just fine. 

The other values I have in ```~/.bash_profile``` are: 
```
OS_TENANT_ID=c0ffeeface7041188ff69da5d3db5c4a
OS_IDENTITY_API_VERSION=2
OS_PASSWORD=supersecret
OS_AUTH_URL=http://api-trial5.client.metacloud.net:5000/v2.0
OS_COMPUTE_API_VERSION=2
OS_USERNAME=myusername
OS_TENANT_NAME=mytenantname
```

The last part I have in the ```provisioners``` section is the script I want to execute to build my image.  This script
is placed in the same directory that the packer file is in so it can be referenced locally.  The script is
available [here on Github](https://raw.githubusercontent.com/vallard/CiscoCloudDayLab1/master/00-Setup/Packer/jenkins_setup.sh)

To run the script I would simply do: 
```
packer build coreos-jenkins.json
```
and away it would go. 

### Troubleshooting and other options

This setup seemed to work pretty well with Metapod OpenStack distributions, but there are other distributions I had
troubles with.  In particular in the latest Liberty release I had problems where I would get ssh timeout errors, even
though I could log into it just fine. These were OpenStack clusters I built by myself with my own hands. 

![These Hands](http://www.quickmeme.com/img/6d/6d81c6fb0767104e2a58dce3f5e3666e5a34c41f407bcac7a3f1c75b0c0be46d.jpg "OpenStack built with these hands")

Running in debug mode helped me locate the problem: 
```
PACKER_LOG=1 PACKER_LOG_PATH=./packer.log packer build --debug coreos-jenkins.json 
```
The output of the log showed me problems with the SSH key.  You see, as part of Packers build process it creates
its own SSH keypair so it can log into the system once created.  

 

{
  "_floating_ip_pool": "nova",
  "_demo1_source_image": "4d723f4e-1cd0-4521-ae77-117a9cd18d24",
  "_floating_ip": "xx.xx.xx.xx",
  "builders" : [{
    "use_floating_ip" : false,
    "type": "openstack",
    "ssh_username": "core", 
    "image_name": "coreos-jenkinslave-{ timestamp }}",
    "source_image" : "48342e8e-e24e-4732-8a56-66d6d29e1931",
    "flavor": "m1.small",
    "networks" : [ "93c562e8-6fca-47fb-bf90-9e7317004cd4" ],
    "security_groups" : ["default"],
    "ssh_keypair_name" : "packerKey",
    "ssh_private_key_file" : "packerKey"
  }
],
  "provisioners": [{
    "type": "shell",
    "script" : "jenkins_setup.sh"
  }]  
}





