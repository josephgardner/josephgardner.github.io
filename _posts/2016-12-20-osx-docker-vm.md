---
layout: post
title: Service Fabric Windows Container
comments: true
---
I've been playing with [Azure Service Fabric](https://azure.microsoft.com/en-us/services/service-fabric/) and testing out 
the [latest preview](https://blogs.msdn.microsoft.com/azureservicefabric/2016/12/15/release-of-sdk-2-4-145-and-runtime-5-4-145-for-windows/) 
with Windows container support. My goal is to deploy a stateless OWIN self-hosted Web API in a container to
service fabric. I saw a cool demo of an application upgrade, so I stole the idea of an API that simply returns a color (blue.) 
Version 2 of the service returns a different color (red), so you can actually watch the rolling upgrade across the cluster
as the color changes from blue to red in several load-balanced clients. 

![rolling upgrade](https://cloud.githubusercontent.com/assets/1297859/21374753/153a4a92-c6f7-11e6-8837-1d1ba4926890.png)

**Program.cs**
```cs
using Microsoft.Owin.Hosting;
using System;
using System.Threading;

namespace ColorPicker
{
    class Program
    {
        static void Main(string[] args)
        {
            string baseAddress = "http://*:80/";

            using (WebApp.Start<Startup>(url: baseAddress))
            {
                Console.WriteLine($"ColorPicker listening at {baseAddress}");
                Thread.Sleep(Timeout.Infinite);
            }
        }
    }
}
```

**Startup.cs**
```cs
using Owin;
using System.Web.Http;

namespace ColorPicker
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            HttpConfiguration config = new HttpConfiguration();
            config.Routes.MapHttpRoute(
                name: "DefaultApi",
                routeTemplate: "api/{controller}/{id}",
                defaults: new { id = RouteParameter.Optional }
            );
            config.Formatters.Remove(config.Formatters.XmlFormatter);
            app.UseWebApi(config);
        }
    }
}
```

**ColorController.cs**
```cs
using System.Web.Http;

namespace ColorPicker
{
    public class ColorController : ApiController
    {
        public string Get()
        {
            return "#0000FF";
        }
    }
}
```
**Dockerfile**
```sh
FROM microsoft/windowsservercore
ADD . /app
WORKDIR /app
ENTRYPOINT ["cmd.exe", "/k", "ColorPicker.exe"]
```

## Docker Registry
The service manifest only lets you specify the name of the container, which means you have to deploy it to a registry. I don't
want to push anything to DockerHub, so instead I decided to give [Docker Registry](https://docs.docker.com/registry/) a try.

I'm using a macbook, so I followed the quick guide to run the registry as a container.

```ps
PS C:\> docker run -d -p 5000:5000 --name registry registry:2
```

I'm running Windows in a VM and tried pushing my Windows docker image to the registry running on the mac host. 

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
### Make it insecure
It turns out the docker client requires TLS by default, but the docs also [explain how to run it insecurely](https://docs.docker.com/registry/insecure/) for testing. To disable security, you have to edit the docker config file on the client, and whitelist the insecure registry you want to connect to.

The instructions tell you to edit the docker file directly, which only applies to \*nix clients. I dug a little deeper into how to configure this for both Docker for Mac and Windows.

#### Docker for Mac
Docker for Mac provisions a HyperKit VM based on Alpine Linux. They do some interesting stuff with git to modify files in the docker app (such as a daemon.json file,) and then reload the VM. However, they have since added an option in the advanced settings dialog to add **insecure registries**.

![mac settings](https://cloud.githubusercontent.com/assets/1297859/21413186/810a81c0-c7c3-11e6-8dd1-40d8ee2e7f7b.png)

If you still wish to connect to the host (for debugging, or other reasons,) you have to use `screen` rather than `ssh`. Found that little nugget on [this blog](https://blog.bennycornelissen.nl/docker-for-mac-neat-fast-and-flawed/). Any changes to the host will be lost whenever the daemon is restarted.

Press enter to get a prompt. Disconnect with ctrl+a d.

```bash
$ screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
```

#### Docker for Windows
Docker for Windows lets you run Linux containers on Windows 10 by provisioning a minimal Linux VM using Hyper-V. I'm not aware of any way to access the host environment, and the stable version of docker does not expose any settings to configure the docker file. 

But I want to push my *windows* container, not a *linux* container. Why would I need Docker for Windows? As luck would have it, I needed to install the beta release in order to get the option to switch between linux and windows containers. The beta settings dialog includes daemon options which lets you set **insecure registries**. This is not currently available in the stable release. 

![windows settings](https://cloud.githubusercontent.com/assets/1297859/21413184/7aea2f66-c7c3-11e6-8596-79e131e23490.png)

Another point worth mentioning is that the docker daemon would not start on my Windows VM when set to use linux containers, which has something to do with nested virtualization (running a VM inside a VM.) Since I don't really care about linux containers at the moment, I simply switched to windows containers, and slowly backed into the bushes.  

![homer scare](https://cloud.githubusercontent.com/assets/1297859/21413862/7a260460-c7c8-11e6-8ffd-aa3f161c141b.gif)

### Let's try again...

Now that our client is configured to trust our insecure registry, let's try the push again.

```ps
PS C:\> docker push 10.211.55.2:5000/colorpicker
The push refers to a repository [10.211.55.2:5000/colorpicker]
242b7a69a21f: Pushed
0281ebf270fd: Pushed
4777122753b8: Pushed
de57d9086f9a: Skipped foreign layer
f358be10862c: Skipped foreign layer
latest: digest: sha256:7d3ff34231abec035813d36f1cfe48c737e15ba9b6db9dd238e5c1f166bcb73d size: 1570
```

Success!
