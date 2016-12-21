I've been playing with [Azure Service Fabric](https://azure.microsoft.com/en-us/services/service-fabric/) and testing out 
the [latest preview](https://blogs.msdn.microsoft.com/azureservicefabric/2016/12/15/release-of-sdk-2-4-145-and-runtime-5-4-145-for-windows/) 
with Windows container support. My goal is to deploy a stateless OWIN self-hosted Web API in a container to
service fabric. I saw a cool demo of an application upgrade, so I stole the idea of an API that simply returns a color (blue.) 
Version 2 of the service returns a different color (red), so you can actually watch the rolling upgrade across the cluster
as the color changes from blue to red in several load-balanced clients. 

![rolling upgrade](https://cloud.githubusercontent.com/assets/1297859/21374753/153a4a92-c6f7-11e6-8837-1d1ba4926890.png)

The service manifest only lets you specify the name of the container, which means you have to deploy it to a registry. I don't
want to push anything to DockerHub, so instead I decided to give [Docker Registry](https://docs.docker.com/registry/) a try.

I'm using a macbook, so I followed the quick guide to run the registry as a container. I'm running Windows in a VM and tried pushing my Windows docker image to the registry running on the mac host. 

```ps
PS C:\> docker build -t colorpicker .
PS C:\> docker tag colorpicker 10.211.55.2:5000/colorpicker
PS C:\> docker push 10.211.55.2:5000/colorpicker
```

This surprisingly worked, but gave me a TLS error:

```ps
The push refers to a repository [10.211.55.2:5000/colorpicker]
Get https://10.211.55.2:5000/v2/: http: server gave HTTP response to HTTPS client
```

It turns out the registry requires a certificate by default, but the docs also [explain how to run it insecurely](https://docs.docker.com/registry/insecure/) for testing. To disable security, you have to edit the docker config file. 

**Docker for Mac** provisions a HyperKit VM based on Alpine Linux. To connect to the host, you have to use `screen` rather than `ssh`. Found that little nugget on [this blog](https://blog.bennycornelissen.nl/docker-for-mac-neat-fast-and-flawed/).

```bash
$ screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
```

Press enter to get a prompt. Disconnect with ctrl+a d.
