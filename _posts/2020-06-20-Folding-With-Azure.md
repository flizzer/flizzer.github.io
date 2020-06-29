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
      - A single Virtual Network (`FoldingAtHomeRG-vnet`) with one subnet (10.1.0.0/24)
      - A single NSG or Network Security Group applied to the subnet that allows for remote access via the Internet (`FoldingAtHomeVMSSAccess-nsg`)
      
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

The NSG rules allow SSH and RDP traffic originating from *only* your public source IP to the subnet where your VM instances reside.  I like using this approach when I don't want to deploy a VPN Gateway or when it's just me accessing my resources.  It's definitely not the most scalable approach should multiple entities need to manage the resources, but for these pet projects, it does the trick.  Without these rules and without configuring an alternative method of access, you won't have access to your VM instances.  Supposedly, you can use the [Azure Serial Console](https://docs.microsoft.com/en-us/azure/virtual-machines/troubleshooting/serial-console-overview) but I was unable to login via this method.  It really isn't meant as a substitute for other access methods either, but YMMV.

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

Obviously, you can customize this to be whatever you like.  Since we're using SSH and we want to keep our credentials out of any template files and source code repos, I have updated the template to reference my public key.  I stored this in my [Azure KeyVault instance](https://azure.microsoft.com/en-us/services/key-vault/) ahead of time, so you'll want to make sure you do the same before trying to deploy the template.  This is awesome, because we can just reference the things we want to keep secure instead of having to copy and paste them around to be fodder for the next hack.

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

- The creation and association of a public IP for each instance is handled in the template under the `networkProfile` section:

```json
"networkProfile": {
                        "copy": [
                            {
                                "name": "networkInterfaceConfigurations",
                                "count": "[length(parameters('networkInterfaceConfigurations'))]",
                                "input": {
                                    "name": "[parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].name]",
                                    "properties": {
                                        "primary": "[parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].primary]",
                                        "enableAcceleratedNetworking": "[parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].enableAcceleratedNetworking]",
                                        "ipConfigurations": [
                                            {
                                                "name": "[concat(parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].name, '-defaultIpConfiguration')]",
                                                "properties": {
                                                    "subnet": {
                                                        "id": "[parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].subnetId]"
                                                    },
                                                    "primary": "[parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].primary]",
                                                    "applicationGatewayBackendAddressPools": "[parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].applicationGatewayBackendAddressPools]",
                                                    "loadBalancerBackendAddressPools": "[parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].loadBalancerBackendAddressPools]",
                                                    "loadBalancerInboundNatPools": "[parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].loadBalancerInboundNatPools]",
                                                    "publicIPAddressConfiguration": "[if( equals( parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].pipName, ''), json('null'), union(json(concat('{\"name\": \"', parameters('networkInterfaceConfigurations')[copyIndex('networkInterfaceConfigurations')].pipName, '\"}')),json('{\"properties\": { \"idleTimeoutInMinutes\": 15}}')))]"
```

If memory serves, the `publicIPAddressConfiguration` key basically just has to be set to a valid name, which we're supplying in the parameters file for `pipName`:

```json
"networkInterfaceConfigurations": {
            "value": [
                {
                    "name": "FoldingAtHomeRG-vnet-nic01",
                    "primary": true,
                    "subnetId": "/subscriptions/<insert your subscription id here>/resourceGroups/FoldingAtHomeRG/providers/Microsoft.Network/virtualNetworks/FoldingAtHomeRG-vnet/subnets/default",
                    "applicationGatewayBackendAddressPools": [],
                    "loadBalancerBackendAddressPools": [],
                    "applicationSecurityGroups": [],
                    "loadBalancerInboundNatPools": [],
                    "enableAcceleratedNetworking": false,
                    "nsgName": "",
                    "nsgId": "",
                    "pipName": "FoldingAtHomeInstance-pip"
```

- Last, but not least for the template, you'll see where I tried to use the same tag `folding@home` for all of the associated resources:
```json
"tags": {
                "application": "folding@home"
            }
```

Even though all of these resources are scoped to the same resource group, using tags can be a helpful way to allow for searching and filtering later on.

I think that about covers the important things I wanted to go over via the template.  I will say that there's a cool VS Code extension that I use to sometimes visualize a template:

![ARM Viewer View](/images/ARMViewerView.png)

It's called [ARM Template Viewer](https://marketplace.visualstudio.com/items?itemName=bencoleman.armview), is authored by Ben Coleman, and is averaging 4.9 out of 5 stars on Visual Studio Marketplace currently.  It's great for when you want to see a graphical representation of your template.  You also can import a parameters file, like the one we're using here, to allow you to click on a resource and see the specifics of the parameters defined.  I highly recommend it and kudos to Ben for his hard work.

## Azure CLI Deployment

So, now that we've got our template and our parameters file all fixed up, it's time to deploy.  For this, I like to use [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest) commands.  You can install it from [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).  In the same repo, you'll find the [deployVMSS.azcli](https://github.com/flizzer/FoldingtAtHomeWithAzure/blob/master/VMSSTemplate/deployVMSS.azcli) file containing the commands we'll need.  Of course, the first thing you must do is login:

```powershell
az login
```

Now that we've gotten that out of the way, let's look at the commands:

- Find which VM skus are supported in which region:

```powershell
az vm list-skus --location eastus2 --size Standard_NV --output table
```
To find if a particular VM series is available in a specific region, you can use some variant of the above command.  This will help avoid errors when you actually deploy.

- Update your KeyVault

```powershell
az keyvault update  --name bhd-key-vault --enabled-for-template-deployment true
```

In the last section I mentioned how referencing secrets from Azure Key Vault is a great thing.  However, apparently, you must explicitly tell your Key Vault instance you want to do this.  Now you can add that public SSH key:

```powershell
az keyvault secret set --vault-name bhd-key-vault --name "ssh-public" --value <insert your public key value>
```

- Validate your deployment

Ok, here's the meat and potatos of all of this.  There's a command we can run to validate our template; think checking for compile-time errors:

```powershell
az deployment group validate `
    --resource-group FoldingAtHomeRG `
    --template-file template.json `
    --parameters parameters.json
```

Notice we specify the RG and then the paths to the template file and associated parameters file respectively.  This will help whittle down any major errors so we can get our files in shape before actually attempting to deploy.

- Deploy

Ok, now that all looks good with our files and we've weeded out as many problems as we can before deployment, here we go:

```powershell
az deployment group create `
  --name FoldingAtHomeDeployment `
  --resource-group FoldingAtHomeRG `
  --template-file template.json `
  --parameters parameters.json
  ```
  
Simply give a name for the deployment and specify the RG, template, and parameters files.  The deployment name is used to reference the deployment.  Hopefully, no errors occur but if they do, think of them as run-time errors.  You might have to tweak your template/parameters file still if you happen to run into any.  If all goes well, you should see the resources appear in your Azure subscription under the `FoldingAtHomeRG` in just a few minutes.  It took maybe 5 minutes or so when I deployed.  Pretty slick!

## Installing and Configuring Folding@home software

Ok, now that we've got some resources to work with, how do we actually participate in the Folding@home project.  I'd like to point out another great [blog post](https://medium.com/@hughxie/run-folding-home-in-an-azure-vm-3d2df989a33e) that was a huge help in getting this going.  Hugh Xie's post was very easy to follow and helped streamline the process.  In fact, the choice for using Ubuntu-based VMs was based on this post after seeing how a few SSH commands could get this going.  In fact, it's so good, I'm not sure I really have a lot to offer other than just follow his guide!  Since you've now got resources deployed, you can skip to step 2.  From there, he'll walk you thru downloading the software needed to connect to Folding@home.  I recommend to use `full` system resources when prompted, and for me, I just chose to contribute anonymously for now.  Maybe I'll create or join a team in the future 🤷‍♂️.  After installing the software you should be able to monitor your VM instances both on the machine via the `tail` and `watch` commands that Hugh gives you as well as via the VMSS page of your subscription in the [Azure portal](https://portal.azure.com).

## Observations So Far

If you've made it this far, I hope you now have Azure resources contributing to the Folding@home project!  If you do, that should make you feel really good.  If you're like me, you've learned a lot through this process and are contributing, perhaps ever so slightly, towards helping humanity as a whole.  So, congratulations!  I did want to just give my observations on what I've seen so far as well.  Really, it's only been a few weeks since I started contributing.  I'm a bit bummed because after I deployed and had this cranking, it only took a couple of days for me to reach the credit limit on my Visual Studio Enterprise subscription ($150/month).  The rest of the time, my subscription has been sitting idle.  It's good that those credits have been used (because they weren't before), but I feel like I could be doing so much more.  I'm also bummed because I couldn't deploy Spot VMs as I've mentioned a few times now.  That really would have knocked the cost down, but there also would have been the potential for those VMs to get kicked out when the capacity was needed, and that would've wasted time as well.  If I were just using a normal PAYGO subscription, I think I could have easily racked up a bill that was at least a couple thousand.  I guess I could play with using a less powerful NV series VM or maybe one without a GPU, but I kind of think if I have to do that then, what's the point?  I've also thought about creating a "Donate" link or trying to get some kind of sponsorship or [Patreon page](https://www.patreon.com).  If you have any ideas, please let me know in the comments below.

All in all though, I really like this solution.  I love the fact that I can tweak and redeploy in just a few minutes.  Using the VMSS allows for flexibility with scaling should I want to do that in the future.  It's pretty elegant and once it's running, it feels pretty "set it and forget it".  Although, if you are on a non-credit based subscription, I would advise creating a budget and then setting budget alerts so you get notified at different thresholds such as 50%, 100% or whenever you'd like to be notified.  That way, "forgetting it" won't mean buying the farm (the server farm maybe, but not the actual...umm...farm...nevermind 😏)

## Next Steps

The next immediate task on the horizon which I would like to do is get this template added to the [Azure QuickStart Templates](https://azure.microsoft.com/en-us/resources/templates/) site. This is a collection of ARM templates for all kinds of workloads and scenarios.  Surprisingly, at least to me, there are no templates for Folding@home workloads.  So, I would very much like to get this added in order to make it more widely available.  From what I've read, this requires a pull request to the Quickstart Templates repo and a bit of standardization. If I can get this done, I'll update this post or maybe even do a new one on the experience.

## Conclusion

I know we've covered a fair amount of territory and you may be thinking "Gosh...there's a lot to get my head around in order to do this."  I would just say, I hope a lot of the heavy lifting has been accounted for via the template.  I certainly have much more to learn regarding ARM Templates, but one doesn't have to be an expert in order to use them.  In light of this, you just need to do three things:

1. Update the parameters file placeholders with your specifics.
2. Validate and Deploy
3. Install the Folding@home software and verify.

That's it!  Using the template as-is should be pretty easy to deploy.  And honestly, your help is needed!  The COVID-19 pandemic is still raging as are these other diseases, and your horsepower would help expedite finding cures.  If you have constructive feedback on the template or ways to make it better, please chime in in the comments below.  Thanks and Happy Folding!
