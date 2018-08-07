# Azure Web App Deployment with Docker Containers

TLDR;
Build and push your Docker image to the Azure Container Registry, then create an Azure Web App to serve the container built from that image.
This does not cover monitoring, failover, database interaction, any external integration, or staged-deployment.
It's only the deployment of a simple one-route web "Hello, World"-type web app.

There's a number of ways to deploy applications into the Azure cloud.
This article focuses on one particular method: deploying to Azure Web Apps with a Docker container.
This article assumes that you have an account with the Azure portal.
If you don't, you should talk to the Cloud Program or your manager.
Also, if you haven't already, you should do yourself a favor and download and install the Azure Command Line Interface (v. 2.0 as of this writing).
The install is exceedingly simple with MacOS X and Homebrew.
You should also have Docker installed on your machine.

Any time I'm mentioning a value that can be variable, it'll be enclosed in angular brackets, `<like this>`.
If you ever screw up and need to delete a resource, the command will be something like `az <resource type> delete --name <resource name> --resource-group <resource group name if it's not a resource group itself>`.

Here are some examples:

- `$ az group delete --name emtech-rg` : deletes a resource group named "emtech-rg" and all of the resources contained within that group
- `$ az vm --name sample-vm --resource-group emtech-vm-rg` : deletes an Azure Virtual Machine named "sample-vm" from the resource group "emtech-vm-rg", but leaves the rest of the resource group intact

## Step-by-step guide

If you're not already logged-in to the Azure CLI, do so with az login.
It should give you a link to log in with your Starbucks account.

Create an Azure Container Registry to house your project's containers, as well as the container(s) that'll go in that registry.

Before you provision any resources in Azure, you must create a Resource Group.
Typically, you want to create a different resource group for different conceptual sets of resources.
Right now, you're going to want a resource group for your project's Container Registry.
Its only job will be to hold your Azure Container Registry instance as well as the containers that will live in that registry.
We separate this from the app itself so that the apps are easy to build/destroy without affecting the containers that they (or other projects) may depend on.
The following command will provisiona Resource Group named "emtech-registry-rg" in the US West region:

```
$ az group create --name emtech-registry-rg --location westus
```

Create your Container Registry within the above resource group.
It can be the one container registry that your whole team uses, or just a container registry for your project.
The registry that you create is effectively a VM whose only purpose is to serve containers.
As such, you need to tell it what type of computer it should be using by giving it the proper SKU, either Standard, Basic, or Premium.
Learn more about what each SKU level involves here.
Make sure that you choose an SKU that can hold at least the full size of your Docker image.
Finally, you want to enable an admin so that you can use the registry name and password to login and push through Docker's own command line tool.
Here's the command to construct a Basic-level registry called "emtech-registry" with admin enabled, sitting in the "emtech-registry-rg" resource group:

```
$ az acr create --name emtechregistry --resource-group emtech-registry-rg --sku Basic --admin-enabled true
```

You'll need your registry's admin password for when you push your Docker container to Azure Container Registry.
Don't worry—it was automatically generated for you when you created the registry.
The following command will allow you to view your registry's admin password. You should probably store it in an environment variable like `ACR_PASS`:

```
$ az acr credential show --name emtechregistry --query "passwords[0].value"
Result
--------------------------------
AbsolutelyFakePassword1234!= 

$ export ACR_PASS='AbsolutelyFakePassword1234!='
```

Log your Docker client into your registry. You'll be using `<your registry name>.azurecr.io` as the place where you login to and provide the password you generated in Step 3.
Note that there are two ways to input your password.
You can either use the `-p` flag to input your plaintext password directly, or you can leave the flag off to have your Docker client prompt you for a password.
Here's both ways of logging your Docker client into an Azure Container Registry:

```
$ docker login emtechregistry.azurecr.io --username emtechregistry --password $ACR_PASS
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
Login Succeeded

$ docker login emtechregistry.azurecr.io # note: the environment variable can't be used here
Username (nmhwRegistry): emtechregistry
Password:
```

If you already have a Dockerfile ready to go you can skip this step.
Create a Dockerfile that will support and run your application.
Be aware of what port it's supposed to expose while running, as you'll be making your Azure Web App aware of it later on.
This repository for a bare-bones Flask (Python) application contains a Dockerfile that'll set up a Python 3.7 environment, install the necessary packages, and expose the running application when the container is run.
That VERY BASIC Dockerfile is reproduced below:

```docker
FROM python:3.7
MAINTAINER Nicholas Hunt-Walker "nhuntwalker@gmail.com"
COPY . /app
WORKDIR /app
RUN pip install --upgrade pip
RUN pip install -r requirements.txt
ENTRYPOINT ["python"]
CMD ["app.py"]
```

If you haven't already, you'll want to build your Docker image locally.
What you'll be doing is pushing your constructed image to the Azure Container Registry, not your repository or Dockerfile.
You can call it whatever you like.
To build the Docker image for the Flask application above, cd into the directory housing its Dockerfile and run

```
$ docker build -t flask-example:latest .
```

You'll get a bunch of output as it sets up an Ubuntu image, sets up all the OS necessaries, sets up Python 3.7, and installs the Python packages in `requirements.txt`.
Tag your Docker image with your registry's URL.
The following command will do that for the Flask image that was just built

```
$ docker tag flask-example emtechregistry.azurecr.io/flask-example:v1
```

Push your tagged Docker image to the Azure Container Registry.
This is of course assuming that you've already logged your Docker client into the Azure Container Registry (see Step 4).
To push the above Docker image into the Azure Container Registry that was created, execute the following command:

```
$ docker push emtechregistry.azurecr.io/flask-example:v1
```

At this point your image is up and running in the Azure Cloud. Woo 🎉🎉🎉🎉!
Now you're going to want to construct your web app to serve the contents of that image to the world.

## Build an Azure Web App with Continuous Deployment from your Docker Container

Because your Azure Web Apps don't live within the space of concern of Docker containers, it's probably a good idea that they get their own resource group.
Of course, you don't have to build it that way, but it does make maintenance a bit easier.
Do yourself a favor and create a new resource group for your upcoming Azure Web App:

```
$ az group --name emtech-webapp-rg --location westus
```

Every Azure Web App needs an Azure App Service Plan.
In a practical sense, your web app's service plan will determine what type of computing resources your web app will consume.
What region does it live in? How many virtual machines will it run on?
How big are those virtual machines?
Does your application scale out (provision more VMs as needed) automatically?
All of these questions will get answered by the type of Service Plan you provision.
For the Flask app, let's keep it really simple and provision an S1-level Linux App Service Plan:

```
$ az appservice plan create --name emtech-service-plan --resource-group emtech-webapp-rg --sku S1 --is-linux
```

Now you want to create the Azure Web App itself.
The critical point in the command that'll follow is the declaration of a container as the deployment mode, as opposed to Github, local Git, or any other method.
Everything else is as you might expect at this point: the app needs a name, needs to be associated with a resource group, and associated with an app service plan.
Here's the command that'll create the Azure Web App from the Azure Container instance we created in the first section:

```
$ az webapp create --name emtech-webapp --resource-group emtech-webapp-rg --plan emtech-service-plan --deployment-container-image-name "emtechregistry.azurecr.io/flask-example:v1"
```

The Web App is constructed but it has yet to be configured.
Without specifying the username, password, and registry, your Web App actually can't access and build your image.
To configure such important parameters for the Web App's deployed container, the following command should do the trick:

```
$ az webapp config container set --name emtech-webapp --resource-group emtech-webapp-rg --docker-registry-server-url https://emtechregistry.azurecr.io --docker-registry-server-user emtechregistry --docker-registry-server-password $ACR_PASS --docker-custom-image-name 'DOCKER|emtechregistry.azurecr.io/flask-example:v1'
```

Almost done!
Step 4 configured the container that the Azure Web App built.
We need to configure the Azure Web App itself to listen to a specific port from the container, and tell it to enable continuous deployment so that when we update our image it updates our Web App.
Our Flask app will, by default, serve on port 5000 within the container.
By making our Web App aware of that, the HTTP response that would come from that port will now be served over port 80 from the VM hosting the web app (or 443 if we're serving over HTTPS).
This command will update those settings for the App:

```
$ az webapp config appsettings set --name emtech-webapp --resource-group emtech-webapp-rg --settings PORT=5000 DOCKER_ENABLE_CI=true
```

It's all set to go! Restart the Web App and enjoy your newly deployed project!

```
$ az webapp restart --name emtech-webapp --resource-group emtech-webapp-rg
```