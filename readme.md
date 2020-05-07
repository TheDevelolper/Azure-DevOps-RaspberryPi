# Creating a Raspberry Pi Azure DevOps Build Agent

## Why? 

I've got a couple of Pi machines sitting around. I wanted to do something useful with them. 

I'm currently collaborating with other developers on personal projects. We're using automated builds to create nuget packaged reusable components. I wanted to make sure that we didn't consume more than the 1800hrs per month free build time. So I thought it would be a good idea to put those Pi's to work!


## Step by Step Guide

### Setup Rasbian Buster Lite

* Download Rasbian Buster Lite (Lite doesn’t have a desktop) - ignore any outdated guides telling you to use Stretch. Get it here: https://www.raspberrypi.org/downloads/raspbian/

* Install Rufus or some image burner, then write the image to your SD card. 

* Pop the SD card in your Pi and turn it on, it will set itself up.


### Setup Wifi and SSH 

Use the following terminal command to setup WiFi and SSH:

``` terminal
> sudo raspi-config
``` 

* Go to network options and setup Wifi
* Go to Interfacing Options and enable SSH

Test SSH by remoting into your RasPi from another machine. 

``` terminal
> ssh pi@<ip address>
```

Now change your password:

``` terminal
> passwd
Enter current password: 
Enter new  password: 

Changing password for pi.
Current password:
New password:
Retype new password:
passwd: password updated successfully
```


### Install Docker

``` terminal
> sudo apt-get update -y && \
sudo apt-get upgrade -y && \
curl -fsSL https://get.docker.com -o get-docker.sh && \
sudo sh get-docker.sh
rm get-docker.sh
```

Now test docker using the hello world docker container: 

``` terminal
> sudo docker run hello-world && docker rm

Hello from Docker!
This message shows that your installation appears to be working correctly...

```

### Install Portainer

Now let's run and install portainer to manage our containers and instances. 

``` terminal
> sudo docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

Wait a few seconds after the container has started then nagivate to `http://<pi's-ip>:9000`

And you will be prompted to set a new password for the portainer admin account. Once this is done click Local on the next screen and you'll see a list of instances (which will only contain local).

Click local and you will get a dashboard where you can manage images and containers using a GUI. 

Now we need to use docker to create an image for the Azure DevOps. 

Create a folder and DockerFile:

``` terminal
> sudo mkdir buildAgent && \
    cd buildAgent && \
    sudo touch Dockerfile && \
    sudo nano Dockerfile
```

Now paste in the following content into the docker file:

``` docker
FROM mcr.microsoft.com/dotnet/core/sdk:3.1
 
# Install curl, wget and git
RUN apt-get update 
RUN apt-get install -y curl wget git
 
# Download compiles vsts-agent
RUN curl https://vstsagentpackage.azureedge.net/agent/2.166.4/vsts-agent-linux-arm-2.166.4.tar.gz -o vsts-agent-linux-arm-2.166.4.tar.gz
RUN mkdir vsts-agent
RUN tar xzf vsts-agent-linux-arm-2.166.4.tar.gz -C ./vsts-agent
 
# install node
RUN curl -sL https://deb.nodesource.com/setup_9.x
RUN apt-get install -y nodejs

COPY vsts.sh .

ENTRYPOINT [ "/bin/bash", "./vsts.sh" ]
```

Now we need to create the vsts.sh script referenced in the Dockerfile above: 

``` terminal
> sudo touch vsts.sh && sudo nano vsts.sh
```
Now paste in the following content

``` bash
#! /bin/bash
./vsts-agent/bin/Agent.Listener configure --unattended --url $VSTS_SERVER_URL --auth PAT --token $VSTS_TOKEN --pool $AGENT_POOL --agent $AGENT_NAME --replace --acceptTeeEula
./vsts-agent/bin/Agent.Listener run
```
> We might need to consider hard coding values here!

Now we can build the docker image:

``` terminal
> sudo docker build -t azure-devops-agent:latest ./ 
```

``` terminal
> sudo docker run -e VSTS_SERVER_URL=https://dev.azure.com/<your username> -e VSTS_TOKEN=<pat token> -e AGENT_NAME=raspi -e AGENT_POOL=RasPi --name azure-build-agent azure-devops-agent:latest ./
```

## Sources: <p>
Original Dockerfile Reference (though I updated this a bit):<br>
https://github.com/Ellerbach/Azure-DevOps-Docker

Images for latest DevOps Images:<br>
https://github.com/dotnet/dotnet-docker/tree/master/3.1/sdk/buster
</p>


