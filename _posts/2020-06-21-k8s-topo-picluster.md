---
layout: post
title:  "Network simulations with k8s-topo on Raspberry Pi"
---

Network simulations on Raspberry Pi cluster
==========================================

This post covers how I set up [k8s-topo](https://github.com/networkop/k8s-topo)
on a Raspberry Pi cluster to perform network simulations. `k8s-topo` is a sweet
project that lets you spin up arbitrary network topologies on Kubernetes
clusters. The router nodes can be cEOS, Quagga, or (soon!) FRR.

Fair warning: this involves a *lot* of source builds. The software industry -
and this is especially true of the k8s / Docker communities - is largely an
amd64 monoculture these days, and our target platform is ARM. Furthermore,
Kubernetes is notoriously complicated and somewhat underdocumented and takes
full advantage of semantic versioning to regularly break its APIs between
versions. Similarly, the Go ecosystem is still a moving target and language
features relied upon by tooling often require recent versions. Finally, anyone
who's worked with Kubernetes before knows that there's really no such thing as
Kubernetes; there's only Kubernetes implementations, each of which have their
own little quirks that like to block you. k3s seems to be one of the better
ones in this regard but there are still quirks.

Because of these factors, and because of the painfully slow build time of large
software on Raspis, this project is a rather involved and will probably take at
least a full day. Nevertheless, with perseverance, we shall prevail.

Prerequisites
-------------

- A k3s cluster of Raspberry Pis, or similar armv7l devices *with hard float
  support*
- Raspbian 10
- `bash` - all of my command lines are specific to `bash` but could be adapted
  to other shells. Also, they're all implied to be root shells.
- Docker, on the master node:

  ```
  apt install docker.io
  ```

- Patience

I cannot emphasize hard float enough. Unfortunately nobody cares about soft
float devices, especially Python. If you are on soft float do not waste your
time. If you do attempt this project on soft float and succeed, I'd love to
hear about it.

Note that while my target device, Pi 3B+, is a 64-bit platform, I have tried to
avoid taking the easy way out and downloading prebuilt arm64 binaries. That is
what got us into this monoculture mess to begin with. I do liberally download
32 bit armv5 binaries though, because some of the tools used can take days to
build on a Raspberry Pi (looking at you protobuf).

Also, before we go any further, add this line to the end of your `.bashrc`:

```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

A lot of tools look at this environment variable to know where the k8s global
config is.

Guide
-----

Best to begin from the top.

End goal: Get `k8s-topo` running.


### Step 1: Install Dependencies

So, the main thing `k8s-topo` needs is a way to configure appropriate network
devices in a Kubernetes-sanctioned way; something to sit on top of the "native"
Kubernetes networking sludge and provide a playground to allow us to create
interfaces accessible to our containers and wire them up. `k8s-topo` supports
two methods of configuring the network resources it needs.

The first is via a "CNI plugin" (Container Network Interface). When k8s is told
that it needs to create some network resources, it first looks at a
predetermined location on the node - typically `/etc/cni/net.d/` - for a config
file(s). This config file tells it which CNI plugins are available and which
one to call. In particular, the config file specifies the path of the plugin
binary, some general k8s-related options, and some binary-specific options to
pass. K8s then calls this binary, passes it the requested state of the complete
cluster network in environment variables (lol), and expects the binary to make
it so. The binary itself typically interacts with the k8s API server to perform
its actions, although it may do so indirectly by way of other tools, or even
with other CNI plugins.

The author of `k8s-topo`, [@networkop](https://github.com/networkop), has built
a CNI plugin called [meshnet-cni](https://github.com/networkop/meshnet-cni).
There is a nice (but slightly outdated) writeup on it
[here](https://networkop.co.uk/post/2018-11-k8s-topo-p1/). Tl;dr it creates
`veth`'s for connectivity between pods on the same node and runs a daemon on
all the nodes to coordinate creation of VXLAN p2p links for cross-node pod
connectivity  When I started on this project, it didn't have support for k8s
1.18, which is what I was using. After opening a GitHub issue asking dumb
questions in which I mentioned this fact, he updated it to support 1.18 (thank
you!). By that time I had already attempted to use the other supported method,
[NetworkServiceMesh](https://networkservicemesh.io/). I got pretty far, but
ultimately ran into some behavior that appears to be either a bug in NSM or a
misconfiguration by me that I was not able to solve. For completeness, I'll
cover my attempts on that front in case you want to try this method out. Maybe
someone reading this will tell me what I did wrong.

`NetworkServiceMesh` does sort of the same thing as `meshnet-cni`, but in a
different way. Since my end goal was really just to get my network sims going I
didn't spend too much time researching what a "Service Mesh" is.

#### Setting Up NetworkServiceMesh

Warning:

> I was not able to get this working. I feel like I got pretty close, so if
> you're adventurous perhaps you can figure out the last piece of the puzzle,
> I would love to know about it.

NSM is deployed with Helm, so we have to set up Helm first. Helm is like a
package manager for k8s.

Download a Helm binary release from the GitHub releases page. NSM only supports
Helm 2, so we'll need to install that first. At time of writing, 2.16.7 is the
latest version, so that's what I used. Make sure to download the `arm` archive.


(Actually shortly after I wrote this, NSM gained support for Helm 3. I didn't
try it since Helm 2 works.)

```
wget https://get.helm.sh/helm-v2.16.7-linux-arm.tar.gz
```

Extract the archive and "install" Helm:

```
tar xvzf helm-v2.16.7-linux-arm.tar.gz
cp linux-arm/helm /usr/local/bin/helm
```

The Helm client is now installed. Next step is to install the backend, called
`tiller`.  k3s is a bit particular regarding permissions, so we need to give
`tiller` its own k8s service account to do stuff with.

```
kubectl -n kube-system create serviceaccount tiller

kubectl create clusterrolebinding tiller \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:tiller
```

References:
https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
https://rancher.com/docs/rancher/v2.x/en/installation/options/helm2/helm-init/

Now we start getting to the x86 monoculture stuff.

The backend for Helm, `tiller`, runs in a container in the k8s cluster itself.
Despite the Rancher docs claiming you can simply `helm init`, in reality, if
you do this Helm downloads and deploys an amd64 `tiller` image. Obviously on
ARM this won't work; it will just crash with `exec format error`. We need an
ARM image. As it turns out, Helm doesn't actually build these, because real
computers are all amd64, right?

Fortunately, someone looking out for us aliens builds ARM images of `tiller`
and publishes them on his personal DockerHub registry. I could write another
post railing against binary-only dependencies downloaded from random people on
the internet, maybe another time. Hopefully his registry is still available
when you're reading this document. If not you'll have to find out how to build
`tiller` for ARM yourself.

So, to initialize Helm with an appropriate backend image:
  
```
helm init --service-account tiller --tiller-image=jessestuart/tiller:v2.16.7
```

*Note the tag*. If you downloaded a later version of Helm 2 earlier, you'll
need to change the tag version to match. The backend version must match the
client version (you can see what you installed with `helm version`).

<hr>

Next we'll have to build NSM ourselves, because they don't provide images for
anything except amd64.

Much of NSM is written in Go, so we need the appropriate Go version. Since Go
is rapidly changing, you might have to tweak the version listed here; at time
of writing, NSM requires Go 1.13. Other things require Go, and 1.13 worked for
everything else I built, so this seems to be a good version.

Install Go 1.13:

```
wget https://dl.google.com/go/go1.13.11.linux-armv6l.tar.gz
tar -C /usr/local -xzf ./go1.13.11.linux-armv6l.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

Clone the NSM repo:

```
git clone https://github.com/networkservicemesh/networkservicemesh.git
cd networkservicemesh
``` 

NSM has two forwarding plane implementations available. One is based on VPP
(the default), the other uses the kernel. The VPP image, naturally, doesn't
support our particular flavor of ARM:

```
root@clusterpi-master:/home/pi/vpp-agent# make prod-image
IMAGE_TAG= ./docker/prod/build.sh
Current architecture (armv7l) is not supported.
```

Fortunately the kernel forwarder does support ARM, so we'll use that one
instead.

Not having found anything in the NSM docs about how to properly build this
thing for ARM, very hacky procedures follow.

By default, the recommended `make` invocation in the NSM docs builds test
images, one of which uses the VPP forwarder, so we need to turn that off. Also
turn the forwarder itself off.

`cd` into your NSM clone and apply the following patches:

```
diff --git a/forwarder/build.mk b/forwarder/build.mk
index 0d24b89f..23c1cac4 100644
--- a/forwarder/build.mk
+++ b/forwarder/build.mk
@@ -12,7 +12,7 @@
 # See the License for the specific language governing permissions and
 # limitations under the License.
 
-forwarder_images = vppagent-forwarder kernel-forwarder
+forwarder_images = kernel-forwarder
```

```diff
diff --git a/test/build.mk b/test/build.mk
index 97354fbf..d1aff0d3 100644
--- a/test/build.mk
+++ b/test/build.mk
@@ -14,7 +14,7 @@
 
 test_apps = $(shell ls ./test/applications/cmd/)
 test_targets = $(addsuffix -build, $(addprefix go-, $(test_apps)))
-test_images = test-common vpp-test-common
+test_images = test-common # vpp-test-common
```

At this point we can build the project. This takes about an hour on my Pi 3B+.

```
make k8s-build
```

In order to work around some other stuff, we'll need raw tarballs of the Docker
images we just built. There's a `make` target to save all those.

```
make k8s-save
```

Your built images are now in your local docker registry, but **k3s doesn't use
that**, it has a private one. If you try to deploy the NSM Helm chart at this
point, you'll end up pulling the amd64 images. These images will then be cached
on every single one of your nodes, and will not be pulled again due to the pull
policy in NSM's pod specs being `IfNotPresent`, leading to much pain.

In case you already did this by mistake, you'll need to log into each node and
run the following to delete the amd64 images from the cache:

```
k3s ctr images list | grep networkservicemesh | cut -d' ' -f1 | xargs k3s ctr images remove
```

We now need a way to make our Kubernetes pods use our locally-built ARM images.
There's two ways of doing this, the correct way and the hacky way. The correct
way is to set up a Docker registry, retag all of the images we just built and
upload them to this registry, reconfigure your entire cluster to use this
registry, and rewrite the Helm charts to point at it instead of the default
`docker.io`. The hacky way is to just inject our images into the image cache on
each node so that the nodes believe they have the latest version of the
upstream images and thus defer to our cached ARM images instead of the upstream
amd64 images.

You could also create a local registry and serve your images from there; I
tried this, ran into some TLS issues, canned it.

*Hacky way*

All the images you saved in `make k8s-save` should be in `build/images` as
`tar` archives.

On each cluster node:
- `scp` the images to your node, like `/tmp/images`
- Run this:

  ```
  for file in ./*; do k3s ctr images import $file; done
  ```

Note you'll have to do this for other images later on, so it's probably better
to just use DockerHub.

*Correct Way*

Retag your built images and push them to your personal DockerHub repository.

Now, at last, you can deploy NSM:

```
SPIRE_ENABLED=false INSECURE=true FORWARDING_PLANE=kernel make helm-install-nsm
```

#### Setting up meshnet-cni

Just like with NetworkServiceMesh, we have to make a lot of adjustments, build
custom images, etc.

Off the bat there's a few places where amd64 binaries are downloaded, so we
need to patch those out.
<finish this bit>

### Step 2 - Disable load balancer

Now that we have our underlay deployed and have manually inserted our
`k8s-topo` images into the node image caches, we're finally ready to deploy
`k8s-topo`. Not!

By default k3s runs a load balancer, which binds ports 80 and 443 on every
node. `k8s-topo` wants those, so if you try to deploy `k8s-topo` now, your pods
will remain in `Pending` because port `80` is in use. Don't bother trying to
`netstat -tln`, port 80 won't show up. But it's still in use.  Kubernetes
logic.

Anyway, we need to turn that shit off:

https://rancher.com/docs/k3s/latest/en/networking/#traefik-ingress-controller
https://github.com/rancher/k3s/issues/1160#issuecomment-561572618

Copied here for posterity:

> 1. Remove traefik helm chart resource: `kubectl -n kube-system delete
>    helmcharts.helm.cattle.io traefik`
> 2. Stop the k3s service: `sudo service k3s stop`
> 3. Edit service file sudo nano `/etc/systemd/system/k3s.service` and add this
>    line to `ExecStart`:
> 
>    `--no-deploy traefik \`
> 
> 4. Reload the service file: `sudo systemctl daemon-reload`
> 5. Remove the manifest file from auto-deploy folder: `sudo rm
>    /var/lib/rancher/k3s/server/manifests/traefik.yaml`
> 6. Start the k3s service: `sudo service k3s start`

Probably a good idea to reboot your master node here for good measure.

And finally, we can deploy `k8s-topo`:
- `kubectl create -f k8s-topo/manifest.yml`

Voila.

### Step 3 - Install k8s-topo

<finish this bit>

### Step 4 - profit

### Usage Examples
