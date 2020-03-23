---
title: Using UniTTY to automate tasks
tags: unitty azure docker
---

Thinking inside, outside and around the box is a big part of my job allows me to do exactly that. I have been exploring stuff related to Azure, Kubernetes and automating certain tasks which the current ways of automation do not allow.

Last week, this quest lead me to two awesome projects on GitHub - [ttyd](https://github.com/tsl0922/ttyd) and [GoTTY](https://github.com/yudai/gotty). Both these projects allow you to turn any command line tool to a web app. This got me started with building something that uses this newfound power to do things in a new way and I came up with [UniTTY](https://github.com/ashisa/unitty).

[UniTTY](https://github.com/ashisa/unitty) is a Linux container image that comes preintalled with _Azure CLI_, _PowerShell Core 7_ and _GoTTY_. You can head over to the project here - [https://github.com/ashisa/unitty](https://github.com/ashisa/unitty) - to read more about _UniTTY_ and learn how to use it because here we will just see how I am using it for automation.

The problem that I want to solve with _UniTTY_ deals with deploying services on Azure. You'd think why is that a big deal? And, you are right! - deploying Azure services and automating them isn't a big deal and there are a lot of ways to do that - ARM templates, Azure CLI, PowerShell, Ansible, Terraform, REST APIs and more but what I need to solve for is that the ISV partner I am working with needs to deploy a set of Azure Services on their customers' subscriptions and configure them uniquely for each customer.

The Azure Application offer on Azure Marketplace comes close but it doesn't work since I also need to register the resource providers on each customer's subscription. Short of sending a script that uses Azure CLI commands or PowerShell cmdlets, nothing really works since it needs to be close to zero administrative work for their customers.

How does _UniTTY_ solve it for me? With the simple matter of converting my shell scripts to a web app and allowing me to send a bunch of arguments using the URL!

Let's take an example where I use _UniTTY_ to create a resource group and virtual machine and send the names of these entities using the URL.

We can accomplish this in two ways - first, we send everything using the URL or we can create a script which can be included with the image by adding the steps to the Dockerfile available in the _UniTTY_'s GitHub repo.

In each case, we need to run the following commands -
```
az login
az group create --name myrg --location southindia
az vm create --name myvm01 --resource-group myrg --image UbuntuLTS
```
To run these commands as a web app, you can run _UniTTY_ using Docker as following -
```
docker run -it -p 8080:8080 docker.io/ashisa/unitty gotty /bin/bash -c "az login && az group create --name myrg --location southindia && az vm create --name myvm01 --resource-group myrg --image UbuntuLTS --admin-username 'vmadmin' --admin-password 'R$ND0MPA%%WD'"
```
Single quotes are needed since there are special characters in the password.

Once you run it and browse to [http://127.0.0.1:8080](http://127.0.0.1:8080), you will see that you are prompted to connect to your Azure subscription which will be followed by the resource group and VM creation steps.

If you want to do this using a script, it would look like the following -
```
#!/bin/bash
ARG=$1
RG=$(echo $ARG |awk '{print $1}')
VM=$(echo $ARG |awk '{print $2}')

if [ "$RG" == "" ] || [ "$VM" == "" ]
then
    echo "syntax: $0 resourcegroup-name vm-name"
    echo 
    echo "Send the argument using this URL - http://127.0.0.1:8080/arg=myrg%20myvm01"
else
    az login
    az group create --name $RG --location southindia
    az vm create --name $VM --resource-group $RG --image UbuntuLTS --admin-username 'vmadmin' --admin-password 'R$ND0MPA%%WD'
fi
```
This script requires two arguments - name of the resource group and the name of the VM and the syntax will look like the following -
```
<scriptname> <resource group name> <vm name>
```
If you decide to use the script route, you will need to add the script to the image and build _UniTTY_ with the following diretives in the Dockerfile -
```
# Downloading the script from GitHub
RUN curl https://raw.githubusercontent.com/ashisa/unitty/master/script-cmd/azurecli-script.sh -o ~/azurecli-script.sh \
  && chmod a+x ~/azurecli-script.sh

# Running the script as CMD
CMD [ "/usr/bin/gotty", "--permit-arguments", "/root/azurecli-script.sh" ]
```
An example is available at [https://github.com/ashisa/unitty/tree/master/script-cmd](https://github.com/ashisa/unitty/tree/master/script-cmd). You can clone the repo and make changes to the script as you wish.

Build the image -
```
docker build -t ashisa/unitty-script .
```
Once you've built the image, you can run it as following -
```
docker run -it -p 8080:8080 ashisa/unitty-script
```
Now you can launch the browser and provide the script arguments using the following URL -

[http://127.0.0.1:8080/arg=myrg%20myvm01](http://127.0.0.1:8080/arg=myrg%20myvm01)

You can use _GoTTY_ options such as restricting how many clients can connect to it and how many times; as well as randomizing the URL and adding username/password to access the URL so that should cover the essentials when it comes to security.

Stay tuned for more things that become easy and possible with _UniTTY_.