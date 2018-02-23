[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](http://makeapullrequest.com)
[![unofficial Google Analytics for GitHub](https://gaforgithub.azurewebsites.net/api?repo=AzureContainerInstancesManagement)](https://github.com/dgkanatsios/gaforgithub)
![](https://img.shields.io/badge/status-beta-orange.svg)

# AzureContainerInstancesManagement

This project allows you to manage Docker containers running on [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/).
Suppose that you want to manage a series of running Docker containers. These containers may be stateful, so classic scaling methods (via Load Balancers etc.) would not work. A classic example is multiplayer game servers, where its server has its own connections to game clients, its own state etc. Another example would be batch-style projects, where each instance would have to deal with a separate set of data. For these kind of purposes, you would need a set of Docker containers being created on demand and deleted when their job is done and they are no longer needed in order to save costs.

## High level overview

Project exposes some Functions/webhooks that can be called to create/delete/get logs Azure Container Instances. There is a Function which can be used to report running/active sessions for each container. These sessions could be game server sessions or just 'remaining work to do'. When we create a new container, it takes some time for it to be created. When it's done and the container is running successfully, our project is notified via an [Event Grid](https://azure.microsoft.com/en-us/services/event-grid/) message. There is also a Function that GETs the running containers, the number of their active jobs/sessions as well as their Public IPs. This can be used to see the load of the running containers.

We have a Function that enables the caller to set the state of a container. This can be used to 'smoothly delete' a running container. Imagine this, at some point in time, we might want to delete a container (probably the existing ones can handle the incoming load). However, we do not want to disrupt existing jobs/sessions running on this particular container, so we do it call this Function to set its state as 'MarkedForDeletion'. Moreover, there is another Function that is called on regular time intervals whose job is to delete containers that are 'MarkedForDeletion' and have no running jobs/sessions on them.

Finally, we suppose that there is an external service that uses our Functions to manage running Docker containers and schedule sessions on them.

## Inspiration
This project was heavily inspired by a similar project that deals with VMs called [AzureGameRoomsScaler](https://github.com/PoisonousJohn/AzureGameRoomsScaler).

## One-click deployment

Click the following button to deploy the project to your Azure subscription:

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdgkanatsios%2FAzureContainerInstancesManagement%2Fmaster%2Fdeploy.json" target="_blank"><img src="http://azuredeploy.net/deploybutton.png"/></a>

This operation will trigger a template deployment of the [deploy.json](deploy.json) ARM template file to your Azure subscription, which will create the necessary Azure resources as well as pull the source code from this repository. As soon as the deployment completes, you need to manually add the Event Subscription webhook manually using the instructions [here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid#create-a-subscription). You can use [this](deploy.eventgridsubscription.json) ARM template, just make sure that you select the correct Resource Group to monitor for events (i.e. the Azure Resource Group where your containers will be created).

## Demo

We've created a couple of demos so that you can test the service, check the details at the [DEMOS.md](DEMOS.md) file.

## Technical details

This project allows you to manage [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/) using [Azure Functions](https://azure.microsoft.com/en-us/services/functions/) and [Event Grid](https://azure.microsoft.com/en-us/services/event-grid/). All operations deal with [Container Groups](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-container-groups), which are the top-level resource in Azure Container Instances. Each Container Group can have X number of containers, a public IP etc. Most Functions are HTTP-triggered unless otherwise noted. Moreover, HTTP-triggered Functions are protected by ['authorization keys'](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook#authorization-keys). 

- **ACICreate**: Creates a new Azure Container Group
- **ACIDelete**: Deletes a Container Group
- **ACIDetails**: Gets details or logs for a Container Group/Container
- **ACIGC**: [Timer triggered](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer), runs every 5' by default, deletes all Container Groups that have no running sessions and have been marked as 'MarkedForDeletion'
- **ACIList**: Returns the details (IP/ number of active sessions) about 'Running' Container Groups
- **ACIMonitor**: [EventGrid triggered](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid), it responds to Event Grid events which occur when a Container Instance resource is created/deleted/changed in the specified Resource Group
- **ACISetSessions**: Sets the running sessions for each Container Group
- **ACISetState**: Sets the state of the Container Group (2 options are 'MarkedForDeletion' and 'Failed')

As mentioned before, the HTTP-triggered Functions are supposed to be called by an external service (for a game, this would be the matchmaking service). The details of all containers are saved in an [Azure Table Storage](https://azure.microsoft.com/en-us/services/storage/tables/) table. For each container, this has information regarding its name (specifically, the container group name), the Resource Group it belongs to, its Public IP Address, its active sessions and its state.

Azure Container Groups that are created can be in one of the below states:

- **Creating**: Container group has just been created
- **Running**: Docker image pulled, public IP (if available) ready, can accept connections/sessions
- **MarkedForDeletion**: We can mark a Container Group as `MarkedForDeletion` so that it will be deleted when a) there are no more active sessions and b) the **ACIGC** Function runs
- **Failed**: When something has gone bad

## Flow

A typical flow of the app goes like this:

1. External service calls `ACICreate`, so a new Container Group is created and is set to `Creating` state
2. As soon as the Event Grid notification comes to `ACIMonitor` function, this means that the Container Group is ready so the `ACIMonitor` function inserts its public IP into Table Storage
3. External service can call `ACIList` to get Container Groups in `Running` state as well as `ACIDetails` to get logs/details about the Container Group
4. External service or the Docker containers themselves can call `ACISetSessions` to set running sessions count on Table Storage
5. External Service can call `ACISetState` to set Container Group’s state as `MarkedForDeletion` when the Container Group is no longer needed
6. Time triggered `ACIGC` (GC: Garbage Collector) will remove unwanted Container Groups (i.e. Container Groups that have 0 active/running sesions and are `MarkedForDeletion`)

![alt text](media/states.jpg "States and Transition")

## FAQ

#### What is this **.deployment** file at the root of the project?
This guides the [Kudu](https://github.com/projectkudu/kudu) engine as to where the source code for the Functions is located, in the GitHub repo. Check [here](https://github.com/projectkudu/kudu/wiki/Customizing-deployments) for details.

#### Why are there 4 ARM templates instead of one?
Indeed, there 4 4 ARM files on the project. They are executed in the following order:
- **deploy.json**: The master template that deploys the other 3
- **deploy.function.json**: Deploys the Azure Function App that contains the Functions of our project
- **deploy.function.config.json**: As we need to set an environment variable that gets the value of our 'ACISetSessions' Function trigger URL, we need to set up this template that executes *after* the deployment of the Azure Function App has completed.
- **deploy.eventgridsubscription.json**: Again, we need to get the Event Grid webhook/URL so we manually run this *after* the deployment of the Azure Function App has completed (and the URL has been internally created).

#### I want to handle more events from Azure Event Grid. Where is the definition of those events?
Check [here](https://docs.microsoft.com/en-us/azure/event-grid/event-schema-resource-groups) for resource group events and [here](https://docs.microsoft.com/en-us/azure/event-grid/event-schema-subscriptions) for subscription-wide events.

#### How can I troubleshoot my Azure Container Instances?
As always, Azure documentation is your friend, check [here](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-troubleshooting).

#### How can I manage Keys for my Functions?
Check [here](https://github.com/Azure/azure-functions-host/wiki/Key-management-API).

#### How can I test the Functions?
Not direct Function testing on this project (yet), however you can see a testing file on `tests\index.js`. To run it, you need to setup an `tests\.env` file with the following variables properly set:

- SUBSCRIPTIONID = ''
- CLIENTID = ''
- CLIENTSECRET = ''
- TENANT = ''
- AZURE_STORAGE_ACCOUNT = ''
- AZURE_STORAGE_ACCESS_KEY = ''

#### How can I monitor Event Grid message delivery?
Check [here](https://docs.microsoft.com/en-us/azure/event-grid/monitor-event-delivery) on Azure Event Grid documentation.

#### What's the exat format of the container groups ARM Template (or, what kind of JSON can I send to ACICreate)?
Check [here](https://docs.microsoft.com/en-us/azure/templates/microsoft.containerinstance/containergroups) for the ARM Template for Container Groups.

#### I need to modify the ARM templates you provide, where can I find more information?
You can check the Azure Resource Manager documentation [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview).