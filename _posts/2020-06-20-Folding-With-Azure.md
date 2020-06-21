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
If you're like me, the whole COVID-19 pandemic has made you think about how each one us can help do our part to get us out of this mess and on to brighter days.  Proper social distancing and following the basic guidelines is no small feat to be sure, but what about contributing to actually finding a cure or vaccine.  If you're not an epidemiologist or Dr. Anthony Fauci, figuring out how you can contribute to those noble causes, might be less obvious.  There actually is a way that you can put your IT expertise and resources to good use towards these goals, though.  

There is a project called [Folding@home](https://foldingathome.org) that uses a distributed computing model to crunch numbers for finding cures for different diseases in humans.  According to their website, "folding" refers to the way proteins are folded by human cells and this is critical for understanding how the human body reacts to certain diseases.  In order for cures to be found, tests and calculations have to be run.  The more resources are donated to perform these calculations, the quicker progress is made.  So, anyone with CPU/GPU cycles to spare can make a very real difference by just donating their unused processing power.

The goal of this post is to walk throuh the steps I took to get an environment up and running in Azure for contributing to the Folding@home project.  I hope it will serve as a relatively easy-to-use guide for doing just that.  It's assumed you already have access to an Azure subscription and the rights to add new resources.  You'll also need to be familiar with the Azure CLI or some other means of deploying resources via an ARM (Azure Resource Manager) template.  I'll give you the commands needed for use with the CLI.  I'd also like to acknowledge Josh Heffner and his excellent [post](https://joshheffner.com/how-to-run-foldinghome-on-azure-spot-vms/) which was an invaluable help to me as a blueprint.  It's a great resource for explaining how to use [Azure Spot VMs](https://azure.microsoft.com/en-us/pricing/spot/) to save costs while participating in Folding@home.  At first, this was my goal as well, but though I tried, I was unable to deploy to Spot VMs in any region.  So, I gave up and just went with traditional VMs.  I'm not sure if this was just due to high-demand for Spot VMs or if it's because they weren't supported in the configuration I was trying.  No worries, if you can get them deployed, you can use them as substitutes for the steps I outline here and you'll end up saving on VM costs in the long run.
