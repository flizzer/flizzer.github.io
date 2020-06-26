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
 1. **ARM template**
 2. **CLI deployment**
 3. **Installing and Configuring Folding@home software**
 4. **Personal mileage with running and costs**
 5. **Links to Quickstart templates and hopefully this template will be added there**

If you're like me, the whole COVID-19 pandemic has made you think about how each one us can help do our part to get us out of this mess and on to brighter days.  Proper social distancing and following the basic guidelines is no small feat to be sure, but what about contributing to actually finding a cure or vaccine.  If you're not an epidemiologist or Dr. Anthony Fauci, figuring out how you can contribute to those noble causes, might be less obvious.  There actually *is* a way that you can put your IT expertise and resources to good use towards these goals, though.  

There is a project called [Folding@home](https://foldingathome.org) that uses a distributed computing model to crunch numbers for finding cures for different diseases in humans.  According to their website, "folding" refers to the way proteins are folded by human cells and this is critical for understanding how the human body reacts to certain diseases.  In order for cures to be found, tests and calculations have to be run.  The more resources are donated to perform these calculations, the quicker progress is made.  So, anyone with CPU/GPU cycles to spare can make a very real difference by just donating their unused processing power.

The goal of this post is to walk throuh the steps I took to get an environment up and running in Azure for contributing to the Folding@home project.  I hope it will serve as a relatively easy-to-use guide for doing just that.  It's assumed you already have access to an Azure subscription and the rights to add new resources.  You'll also need to be familiar with the Azure CLI or some other means of deploying resources via an ARM (Azure Resource Manager) template.  I'll give you the commands needed for use with the CLI.  I'd also like to acknowledge Josh Heffner and his excellent [post](https://joshheffner.com/how-to-run-foldinghome-on-azure-spot-vms/) which was an invaluable help to me as a blueprint.  It's a great resource for explaining how to use [Azure Spot VMs](https://azure.microsoft.com/en-us/pricing/spot/) to save costs while participating in Folding@home.  At first, this was my goal as well, but though I tried, I was unable to deploy Spot VMs in any region.  So, I gave up and just went with traditional VMs.  I'm not sure if this was just due to high-demand for Spot VMs or if it's because they weren't supported in the configuration I was trying.  No worries, if you can get them deployed, you can use them as substitutes for the steps I outline here and you'll end up saving on VM costs in the long run.  If you're successful, please leave a comment below!  Maybe I didn't quite hold my mouth right or something 😜.  Let's get started!

As mentioned before, we're going to be using an ARM template to describe the resources needed and then deploy them.  The main resources we're going to be deploying are:

   - A single resource group (`FoldingAtHomeRG` in the `southcentralus` region) that contains all the necessary resources including:
      - A single VMSS or VM Scale Set (`FoldingAtHomeVMSS`) containing:
         - 2 identical VM instances, each configured with a single NIC (`FoldingAtHomeVMSS_0` and `FoldingAtHomeVMSS_1` respectively)
         - A locally-redundant diagnostic storage account (`foldingathomergdiag166` *Please forgive the random naming here...*)
      - A single Virtual Network with one subnet (`FoldingAtHomeRG-vnet`)
      - A single NSG applied to the subnet that allows for remote access (`FoldingAtHomeVMSSAccess-nsg`)
      
An ARM template is great for this scenario.  It allows us to again, *describe*, the resources we want to deploy, and then tweak and repeat until we're satisfied.  In our case, we have a template file and an associated parameters file.  You can find these and the CLI commands we'll use for deploying in my [FoldingAtHomeWithAzure](https://github.com/flizzer/FoldingtAtHomeWithAzure/tree/master) repo.  The template is essentially the baseline code that indicates the resources and the parameters file lets you, you guessed it, parameterize any specifics.  For instance, if you want to deploy 4 instances instead of the 2 I have configured, you can adjust the value for the `InstanceCount` parameter to your liking:

```json
"instanceCount": {
            "value": "4"
        },
```

This really helps automate things instead of pointing-and-clicking through the portal.  I've found that it does pay to go through the portal and configure what you want first though.  Then you can dowload a template to serve as a baseline.  

I won't go through the template line-by-line, but I do want to call out some specific areas:

- The template is currently set to use [`Standard_NV6`](https://docs.microsoft.com/en-us/azure/virtual-machines/nv-series) VMs.  As the link says, these use GPUs and to crunch the most numbers when doing those folding calculations, you need to use something that offloads the processing to a GPU.

-









