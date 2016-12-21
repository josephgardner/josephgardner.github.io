I've been playing with [Azure Service Fabric](https://azure.microsoft.com/en-us/services/service-fabric/) and testing out 
the [latest preview](https://blogs.msdn.microsoft.com/azureservicefabric/2016/12/15/release-of-sdk-2-4-145-and-runtime-5-4-145-for-windows/) 
with Windows container support. My goal is to deploy a stateless OWIN self-hosted Web API in a container to
service fabric. I saw a cool demo of an application upgrade, so I stole the idea of an API that simply returns a color (blue.) 
Version 2 of the service returns a different color (red), so you can actually watch the rolling upgrade across the cluster
as the color changes from blue to red in several load-balanced clients. 

The service manifest only lets you specify the name of the container, which means you have to deploy it to a registry. I don't
want to push anything to DockerHub, so instead I decided to give [Docker Registry](https://docs.docker.com/registry/) a try.

I have a macbook, and am running windows in a VM. So I followed the quick guide to run the registry as a container, and tried
pushing my Windows docker image to the registry running on the mac host. 

```ps
PS C:\> docker build -t colorpicker .
PS C:\> docker tag colorpicker 10.211.55.2:5000/colorpicker
PS C:\> docker push 10.211.55.2:5000/colorpicker
```
