---
layout: post
title: Folding At Home With Your Spare Azure Credits
description: How to help fight COVID-19 with your spare Azure resources
author: Brian Davis
categories: how-to 
tags: folding@home
date: 2020-06-20
comments: true
published: false
---

# Table Of Contents
 1. **Introduction**
 2. **ARM template**
 2. **CLI deployment**
 3. **Installing and Configuring Folding@home software**
 4. **Personal mileage with running and costs**
 5. **Links to Quickstart templates and hopefully this template will be added there**
 
 ## Introduction

If you're like me, the whole COVID-19 pandemic has made you think about how each one us can help do our part to get us out of this mess and on to brighter days.  Proper social distancing and following the basic guidelines is no small feat to be sure, but what about contributing to actually finding a cure or vaccine.  If you're not an epidemiologist or Dr. Anthony Fauci, figuring out how you can contribute to those noble causes, might be less obvious.  There actually *is* a way that you can put your IT expertise and resources to good use towards these goals, though.  

There is a project called [Folding@home](https://foldingathome.org) that uses a distributed computing model to crunch numbers for finding cures for different diseases in humans.  According to their website, "folding" refers to the way proteins are folded by human cells and this is critical for understanding how the human body reacts to certain diseases.  In order for cures to be found, tests and calculations have to be run.  The more resources are donated to perform these calculations, the quicker progress is made.  So, anyone with CPU/GPU cycles to spare can make a very real difference by just donating their unused processing power.

The goal of this post is to walk throuh the steps I took to get an environment up and running in Azure for contributing to the Folding@home project.  I hope it will serve as a relatively easy-to-use guide for doing just that.  It's assumed you already have access to an Azure subscription and the rights to add new resources.  You'll also need to be familiar with the Azure CLI or some other means of deploying resources via an [ARM (Azure Resource Manager) template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview).  I'll give you the commands needed for use with the CLI.  I'd also like to acknowledge Josh Heffner and his excellent [post](https://joshheffner.com/how-to-run-foldinghome-on-azure-spot-vms/) which was an invaluable help to me as a blueprint.  It's a great resource for explaining how to use [Azure Spot VMs](https://azure.microsoft.com/en-us/pricing/spot/) to save costs while participating in Folding@home.  At first, this was my goal as well, but though I tried, I was unable to deploy Spot VMs in any region.  So, I gave up and just went with traditional VMs.  I'm not sure if this was just due to high-demand for Spot VMs or if it's because they weren't supported in the configuration I was trying.  No worries, if you can get them deployed, you can use them as substitutes for the steps I outline here and you'll end up saving on VM costs in the long run.  If you're successful, please leave a comment below!  Maybe I didn't quite hold my mouth right or something 😜.  Let's get started!

## ARM Template

As mentioned before, we're going to be using an ARM template to describe the resources needed and then deploy them.  The main resources we're going to be deploying are:

   - A single resource group (`FoldingAtHomeRG` in the `southcentralus` region) that contains all the necessary resources including:
      - A single VMSS or VM Scale Set (`FoldingAtHomeVMSS`) containing:
         - 2 identical Ubuntu Linux 18.04-LTS VM instances, each configured with a single NIC (`FoldingAtHomeVMSS_0` and `FoldingAtHomeVMSS_1` respectively) and each configured with a public IP for ease of remote administration
         - A locally-redundant diagnostic storage account used for VM boot diagnostics (`foldingathomergdiag166` *Please forgive the random naming here...*)
      - A single Virtual Network with one subnet (`FoldingAtHomeRG-vnet`)
      - A single NSG or Network Security Group applied to the subnet that allows for remote access (`FoldingAtHomeVMSSAccess-nsg`)
      
An ARM template is great for this scenario.  It allows us to again, *describe*, the resources we want to deploy, and then tweak and repeat until we're satisfied.  In our case, we have a template file and an associated parameters file.  You can find these and the CLI commands we'll use for deploying in my [FoldingAtHomeWithAzure](https://github.com/flizzer/FoldingtAtHomeWithAzure/tree/master) repo.  The template is essentially the baseline code that indicates the resources and the parameters file lets you, you guessed it, parameterize any specifics.  For instance, if you want to deploy 4 instances instead of the 2 I have configured, you can adjust the value for the `InstanceCount` parameter to your liking:

```json
"instanceCount": {
            "value": "4"
        },
```

This really helps automate things instead of pointing-and-clicking through the portal.  I've found that it does pay to go through the portal and configure what you want first though.  Then you can dowload a template to serve as a starting point.  

I won't go through the template line-by-line, but I do want to call out some important areas:

- The template is currently set to use [`Standard_NV6`](https://docs.microsoft.com/en-us/azure/virtual-machines/nv-series) VMs for the VMSS instances:

```json
"instanceSize": {
            "value": "Standard_NV6"
        },
```

As the link indicates, these use GPUs and, to crunch the most numbers when doing those folding calculations, you need to use something that offloads the processing to a GPU.  To go along with the VM `instanceSize`, you can also specify a `priority`:

```json
"priority": {
            "value": "Regular"
        },
```

If you wanted to try and use Azure Spot VMs as mentioned a few paragraphs ago, you would set this value to `Spot`.

- There are several places in the parameters file, where you'll need to substitute `<insert your subscription id here>` with your actual subscription value.  For example, when specifying the `virtualNetworkId`, you'll need to use a valid subscription value:

```json
"virtualNetworkId": {
            "value": "/subscriptions/<insert your subscription id here>/resourceGroups/FoldingAtHomeRG/providers/Microsoft.Network/virtualNetworks/FoldingAtHomeRG-vnet"
        },
```

- As mentioned above and per security best practices, an NSG is deployed and associated at the subnet level.  This way any devices deployed to that subnet are governed by the same rules.  It's a much more scalable approach than trying to associate these with each VM's NIC, even if we are using an ARM template to automate the deployment.  Much as with the subscription id, you'll need to substitute valid values for your public source IP for the NSG definitions:

```json
"rules": [
                        {
                            "name": "SSH",
                            "properties": {
                                "protocol": "TCP",
                                "sourcePortRange": "*",
                                "destinationPortRange": "22",
                                "sourceAddressPrefix": "<insert your public source IP here>",
                                "destinationAddressPrefix": "*",
                                "access": "Allow",
                                "priority": 300,
                                "direction": "Inbound",
                                "sourcePortRanges": [],
                                "destinationPortRanges": [],
                                "sourceAddressPrefixes": [],
                                "destinationAddressPrefixes": []
                            }
                        },
                        {
                            "name": "RDP",
                            "properties": {
                                "protocol": "*",
                                "sourcePortRange": "*",
                                "destinationPortRange": "3389",
                                "sourceAddressPrefix": "<insert your public source IP here>", 
                                "destinationAddressPrefix": "*",
                                "access": "Allow",
                                "priority": 310,
                                "direction": "Inbound",
                                "sourcePortRanges": [],
                                "destinationPortRanges": [],
                                "sourceAddressPrefixes": [],
                                "destinationAddressPrefixes": []
                            }
                        }
                    ]
```

The NSG rules allow SSH and RDP traffic originating from *only* your public source IP to the subnet where your VM instances reside.  I like using this approach when I don't want to deploy a VPN Gateway or when it's just me accessing my resources.  It's definitely not the most scalable approach should multiple entities need to manage the resources, but for these pet projects, it does the trick.  Without these rules and without configuring an alternative method of access, you won't have access to your VM instances.  Supposedly, you can use the [Azure Serial Console](https://docs.microsoft.com/en-us/azure/virtual-machines/troubleshooting/serial-console-overview) but I was unable to login via this method.  It really isn't meant as a substitute for other access methods, but YMMV.

The username of the administration account is set to be `foldingadmin`:

```json
"adminUsername": {
            "value": "foldingadmin"
        },
        "adminPublicKey": {
            "reference": {
                "keyVault": {
                "id": "/subscriptions/<insert your subscription id here>/resourceGroups/bhd-key-vault-RG/providers/Microsoft.KeyVault/vaults/bhd-key-vault"
                },
                "secretName": "ssh-public"
              }
        },
```

Obviously, you can customize this to be whatever you like.  Since we're using SSH and we want to keep our credentials out of any template files and source code repos, I have updated the template to reference my public key.  I stored this in my [Azure KeyVault instance](https://azure.microsoft.com/en-us/services/key-vault/) ahead of time, so you'll want to make sure you do the same before trying to deploy the template.  This is awesome, because we can just reference the things we want to keep secure instead of having to copy and paste them around so they can be fodder for the next hack.

- The OS choice is actually set in the template and not in the parameters file.  Take a look:

```json
"virtualMachineProfile": {
                    "storageProfile": {
                        "osDisk": {
                            "createOption": "fromImage",
                            "caching": "ReadWrite",
                            "managedDisk": {
                                "storageAccountType": "[parameters('osDiskType')]"
                            }
                        },
                        "imageReference": {
                            "publisher": "Canonical",
                            "offer": "UbuntuServer",
                            "sku": "18.04-LTS",
                            "version": "latest"
                        }
                    },
```
You can see here that we're creating this from an image and with the `18.04-LTS` sku.

I think that about covers the important things I wanted to go over via the template.  I will say that there's a cool VS Code extension that I use to sometimes visualize a template:

![ARM Viewer View](/images/ARMViewerView.png)

It's called [ARM Template Viewer](https://marketplace.visualstudio.com/items?itemName=bencoleman.armview), is authored by Ben Coleman, and is averaging 4.9 out of 5 stars on Visual Studio Marketplace currently.  It's great for when you want to see a graphical representation of your template.  You also can import a parameters file, like the one we're using here, to allow you to click on a resource and see the specifics of the parameters defined.  I highly recommend it and kudos to Ben for his hard work.

