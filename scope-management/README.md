# Blog: Manage the scope of Istio resources in a multitenant Kubernetes cluster 


# Introduction

[Istio](https://istio.io/) resources affecting network traffic, as [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/), [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/), [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/) apply by default to the entire cluster, this can lead to unwanted behavior, especially in case of multitenant clusters where different teams managing different applications have the rights to create the above resources in their assigned namespace. In this case the traffic management rules defined in a namespace can negatively affect the other workloads running in different namespaces.

In this article you will see 3 ways to limit this impact and set some guardrails to Istio resources scope with an incrementally opinionated approach: from setting the default configuration of the above resources so they apply only to the namespace they are created in, to using [Open Policy Agent (OPA) Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/) to allow only the creation of resources that apply to their same namespace and also directly mutate them to do that.


# What you’ll need

The examples in this article are performed using a [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/docs) cluster with [managed Anthos Service Mesh (ASM)](https://cloud.google.com/service-mesh/docs/overview#managed_anthos_service_mesh) and[ Policy Controller](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller). 

Since all the tasks are based on standard [Kubernetes](https://kubernetes.io/), [Istio](https://istio.io/) and [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/) APIs, these instructions are generally applicable to any standard implementation of the above software projects but some implementation details may differ. 

At the minimum you’ll need a Kubernetes cluster with Istio and OPA Gatekeeper installed.

The assets used in this article are available in this Github repository: [https://github.com/ggalloro/istio-workshop](https://github.com/ggalloro/istio-workshop) 


# The default Istio behavior

When you create an Istio VirtualService, ServiceEntry or DestinationRule resource on a Kubernetes cluster their configuration impacts all the traffic inside the mesh, by default all the traffic from pods that are part of the mesh obey to the above configuration. Let’s see an example of this.

Create 2 namespaces in your cluster to host 2 example applications, imagine these namespaces would be assigned to two different applications teams in a multitenant cluster. Let’s call the namespaces app1 and app2:


```
    kubectl create ns app1
    kubectl create ns app2
```


Label your namespaces to enable automatic Istio proxy injection, in case of managed Anthos Service Mesh we will use the [isito.io/rev](isito.io/rev) label with the asm-managed value:


```
    kubectl label namespace app1 istio-injection- istio.io/rev=asm-managed \
    --overwrite
    kubectl label namespace app2  istio-injection- istio.io/rev=asm-managed \
    --overwrite
```


Clone the repository containing the assets we will use and move inside its local folder:


```
    git clone https://github.com/ggalloro/istio-workshop && cd istio-workshop
```


Deploy the sleep sample application, that we will use to test traffic behavior, to the 2 namespaces you created: 


```
    kubectl -n app1 apply -f sleep.yaml
    kubectl -n app2 apply -f sleep.yaml
```


In Istio (and ASM) you can manage access to external services using the [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/) resource, this resource lets you add entries for these services into Istio’s internal service registry so that traffic to them can be directed and observed as if they were part of the mesh. 

Explore the ServiceEntry resource we will use: open, with an editor of your choice, the file [scope-management/httpbin-ext-tlsorign.yaml](https://github.com/ggalloro/istio-workshop/blob/main/scope-management/httpbin-ext-tlsorigin.yaml). This creates an entry for the [httpbin.org](httpbin.org) external service (a public endpoint that you can point to to test http requests) and instructs the proxy to redirect to port 443 the requests that the originating pod will make to port 80.


```
    apiVersion: networking.istio.io/v1alpha3
    kind: ServiceEntry
    metadata:
      name: httpbin-ext
    spec:
      hosts:
      - httpbin.org
      ports:
      - number: 80
        name: http
        protocol: HTTP
        targetPort: 443
      - number: 443
        name: https
        protocol: HTTPS
      resolution: DNS
      location: MESH_EXTERNAL
```


Explore the [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/) resource we will use: open, with an editor of your choice, the file [scope-management/httpbin-destrule.yaml](https://github.com/ggalloro/istio-workshop/blob/main/scope-management/httpbin-destrule.yaml). This creates a DestinationRule performing TLS origination for HTTP requests that the application makes on port 80


```
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: httpbin.org
    spec:
      host: httpbin.org
      trafficPolicy:
        portLevelSettings:
        - port:
            number: 80
          tls:
            mode: SIMPLE
```


Let’s apply our ServiceEntry and our DestinationRule only in the app1 namespace so we can experiment how this impacts also the app2 namespace with Istio default settings, type the following commands from your local clone of the repo:


```
kubectl -n app1 apply -f scope-management/httpbin-ext-tlsorigin.yaml
kubectl -n app1 apply -f scope-management/httpbin-destrule.yaml
```


Now let’s try to connect to [httpbin.org](httpbin.org) to see the effect of the above configuration.

First set the APP1_POD environment variable to the name of your sleep pod in the app1 namespace


```
export APP1_POD=$(kubectl -n app1 get pod -l app=sleep -o jsonpath='{.items..metadata.name}')
```


Then set APP2_POD environment variable to the name of your sleep pod in the app2  namespace


```
export APP2_POD=$(kubectl -n app2 get pod -l app=sleep -o jsonpath='{.items..metadata.name}')
```


Now let’s make an http request from a the sleep pod in in the app1 namespace to [httpbin.org/anything](httpbin.org/anything) (this path basically returns everything that is in the request):


```
kubectl -n app1 exec -it "$APP1_POD" -c sleep -- curl http://httpbin.org/anything
```


You should get an output containing all the headers in the request, at the end of the response you can see that the url in the request that [httpbin.org](httpbin.org) gets is using the https scheme in place of the http that you put in the request, this is because the ServiceEntry you created above instructs the Istio proxy to redirect to port 443 the requests that have been originally made to port 80 and the DestinationRule instructs the proxy to originate TLS traffic to [httpbin.org](httpbin.org):

…


```
 "url": "https://httpbin.org/anything"
```


Now try to do the same from the app2 namespace


```
kubectl -n app2 exec -it "$APP2_POD" -c sleep -- curl http://httpbin.org/anything
```


You can see that you get a similar output with the same url:

…


```
 "url": "https://httpbin.org/anything"
```


This is because the ServiceEntry and DestinationRule resources apply by default to the whole mesh. In some cases, in multitenant clusters, this behavior can negatively impact the application deployed in other namespaces so you want to narrow the scope of these resources.


# How each application team can set resources scope

Let’s see how a resource creator can set a specific scope to his resources.

In the Istio ServiceEntry, DestinationRule and VirtualService resources you can set the exportTo: field to specify a list of namespaces to which the resource is exported, if no namespaces are specified then the resource is exported to all namespaces by default, this is the behavior we experienced above.

The value “.” defines an export to the resource same namespace while the value “*” defines an export to all namespaces. 

Let’s see how the application team for app1 can narrow the scope of their resources.

In the [scope-management](https://github.com/ggalloro/istio-workshop/tree/main/scope-management) folder of the repo there are also variations of our resource manifests, the ones with the -samens suffix in the file name add the field exportTo: with the value “.” , while in the ones with the -allns suffix the value is “*”. Let’s apply the ones with exportTo: - “.” to narrow the scope of the the resources in the app1 namespace:


```
kubectl -n app1 apply -f scope-management/httpbin-ext-tlsorigin-samens.yaml
kubectl -n app1 apply -f scope-management/httpbin-destrule-samens.yaml
```


Let’s try again to make a request to [httpbin.org](httpbin.org) from the app1 namespace


```
kubectl -n app1 exec -it "$APP1_POD" -c sleep -- curl http://httpbin.org/anything
```


The url in the request still uses the https scheme, this is expected since app1 is where the ServiceEntry and DestinationRule have been created.

Let’s try the same from the app2 namespace:


```
kubectl -n app2 exec -it "$APP2_POD" -c sleep -- curl http://httpbin.org/anything
```


In that case you will see that the url in the request uses the http scheme, because the ServiceEntry and DestinationRule are not exported to the app2 namespace:

…


```
"url": "http://httpbin.org/anything"
```


What we saw would be enough if you can rely exclusively on each application team to set the right configuration on their resources to not negatively impact other application teams.

If you, instead, need to centrally manage the scope of Istio resources configuration, you will see possible solutions with a gradually opinionated approach in the following sections.

To prepare for the following steps, let’s delete the Istio resources we previously created:


```
kubectl -n app1 delete serviceentry httpbin-ext
kubectl -n app1 delete destinationrule httpbin.org
```



# Centrally manage Istio resources scope


## Option 1: Change the default scope in Istio mesh config 

The first thing you can do is to change the default scope of the traffic management resources in the mesh configuration using the following [global mesh options](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/):


```
    defaultServiceExportTo: [.]
    defaultDestinationRuleExportTo: [.]
    defaultVirtualServiceExportTo: [.]
```


This will apply to the whole mesh and will set the mesh default so when a DestinationRule, ServiceEntry or VirtualService are created, even with no exportTo: field set, it will be exported only to its namespace.

The resource creator will still have the possibility to set the exportTo: field to the value he prefers.

To change the mesh configuration in managed ASM you must modify the config map resource named istio-_release-channel_ (where _release-channel_ is your ASM release channel: asm-managed, asm-managed-stable, or asm-managed-rapid) present in the istio-system namespace, let’s do that using the [scope-management/configmap-default-samens.yaml](https://github.com/ggalloro/istio-workshop/blob/main/scope-management/configmap-default-samens.yaml) manifest in the repo:


```
kubectl apply -f scope-management/configmap-default-samens.yaml
```


Run the following command to check that the options have been applied (change the name of the configmap if you have an ASM release channel different than asm-managed):


```
kubectl -n istio-system describe configmap istio-asm-managed
```


You will see an output similar to the following:


```
Name:         istio-asm-managed
Namespace:    istio-system
Labels:       <none>
Annotations:  <none>

Data
====
mesh:
----
defaultServiceExportTo: [.]
defaultDestinationRuleExportTo: [.]
defaultVirtualServiceExportTo: [.]

BinaryData
====

Events:  <none>
```


If you’re not using ASM, follow the instruction for your platform to set [Global Mesh Options](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/), in Istio OSS you will probably use the istioctl install command. 

Now let’s create again the resources in the app1 namespace, with no exportTo: field, as we did when we tested
[the default Istio behavior](#heading=h.rp5rr21us3pf):


```
kubectl -n app1 apply -f scope-management/httpbin-ext-tlsorigin.yaml
kubectl -n app1 apply -f scope-management/httpbin-destrule.yaml
```


Let’s make a request to [httpbin.org](httpbin.org) from app1 namespace:


```
kubectl -n app1 exec -it "$APP1_POD" -c sleep -- curl http://httpbin.org/anything
```


The url in the request still uses the https scheme as programmed by the resources  created in the app1 namespace.

Let’s try the same from the app2 namespace:


```
kubectl -n app2 exec -it "$APP2_POD" -c sleep -- curl http://httpbin.org/anything
```


In that case you will see that the url in the request uses the http scheme, because the ServiceEntry and DestinationRule are not exported to the app2 namespace even if we didn’t set the exportTo: field, this happens because we changed the default mesh behavior.

…


```
"url": "http://httpbin.org/anything"
```


This option is helpful to prevent unwanted impact if the resource creator doesn’t set the exportTo: field, but it doesn’t force any configuration, the resource creator could still set exportTo: to “*” and export resource configurations to all the namespaces.

In the following sections we will see 2 more prescriptive approaches to set boundaries to Istio resources. 

To prepare for the following steps, let’s delete the Istio resources you previously created:


```
kubectl -n app1 delete serviceentry httpbin-ext
kubectl -n app1 delete destinationrule httpbin.org
```



## Option 2: Set an OPA Gatekeeper constraint to allow only resources exported to their namespace

This option uses an OPA Gatekeeper constraint named istio-same-ns ([scope-management/constraint-samens.yaml](https://github.com/ggalloro/istio-workshop/blob/main/scope-management/constraint-samens.yaml)), based on a constraint template named IstioSameNS ([scope-management/template-samens.yaml](https://github.com/ggalloro/istio-workshop/blob/main/scope-management/template-samens.yaml)) to admit, in your cluster, only VirtualServices, ServiceEntry and DestinationRule resources with the exportTo: field set to their local namespaces. 

You can explore the [scope-management/template-samens.yaml](https://github.com/ggalloro/istio-workshop/blob/main/scope-management/template-samens.yaml) manifest to see the template  code.

To perform the following tasks you need OPA Gatekeeper installed in your cluster, the steps below are executed on a cluster using [Policy Controller](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller), a Google managed Kubernetes [dynamic admission controller](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) that checks, audits, and enforces the compliance of your clusters against centrally defined policies, based on the Open Policy Agent (OPA) Gatekeeper.

To use the istio-same-ns constraint, you first need to install the IstioSameNS constraint template it is based on, let’s do that:


```
kubectl apply -f scope-management/template-samens.yaml
```


Now let’s install the istio-same-ns constraint:


```
kubectl apply -f scope-management/constraint-samens.yaml
```


After we have our constraint set up, let’s try to create a ServiceEntry resource with no exportTo: field set as we did when we tested [the default Istio behavior](#heading=h.rp5rr21us3pf):


```
kubectl -n app1 apply -f scope-management/httpbin-ext-tlsorigin.yaml

Error from server (Forbidden): error when creating "scope-management/httpbin-ext-tlsorigin.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [istio-same-ns] spec.exportTo must be set
```


You will get the same result with the DestinationRule:


```
kubectl -n app1 apply -f scope-management/httpbin-destrule.yaml

Error from server (Forbidden): error when creating "scope-management/httpbin-destrule.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [istio-same-ns] spec.exportTo must be set
```


The constraint is checking if the exportTo: field is set on the resources, since it’s not, it is blocking the resource creation.

Let’s try with other resource manifests, available in the repository, with exportTo: set to “*” (all namespaces):


```
kubectl -n app1 apply -f scope-management/httpbin-ext-tlsorigin-allns.yaml
```


This attempt is blocked as well:


```
Error from server (Forbidden): error when creating "scope-management/httpbin-ext-tlsorigin-allns.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [istio-same-ns] spec.exportTo must be set to same namespace [.]
```


Because even if exportTo: is set it’s not set to the only allowed value that is “.” (same namespace).

You will get the same result with the DestinationRule:


```
kubectl -n app1 apply -f scope-management/httpbin-destrule-allns.yaml

Error from server (Forbidden): error when creating "scope-management/httpbin-destrule-allns.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [istio-same-ns] spec.exportTo must be set to same namespace [.]
```


Now let’s try to apply the manifests using the only allowed configuration, exportTo: set to “.” (same namespace):


```
kubectl -n app1 apply -f scope-management/httpbin-ext-tlsorigin-samens.yaml
kubectl -n app1 apply -f scope-management/httpbin-destrule-samens.yaml
```


In this case the creation should be successful:


```
serviceentry.networking.istio.io/httpbin-ext created
destinationrule.networking.istio.io/httpbin.org created
```


If you test the traffic behavior, you will see that Istio TLS origination and redirection will apply only to traffic originating from namespace app1 as in the previous section, with the difference that, with the above constraints, you are not allowing the creation of any resource that has an impact outside its own namespace.

In the next section we will make a step ahead and will also directly mutate the resources at creation.

To prepare for the following section, let’s delete the Istio resources we previously created:


```
kubectl -n app1 delete serviceentry httpbin-ext
kubectl -n app1 delete destinationrule httpbin.org
```



## Option ~~3~~ 2+: Set an OPA Gatekeeper constraint to mutate the resources 

In this section we will add, to the constraint we used above, an OPA Gatekeeper mutation named istio-same-ns available in [scope-management/mutation-istio-samens.yaml](https://github.com/ggalloro/istio-workshop/blob/main/scope-management/mutation-istio-samens.yaml) that will also directly mutate the resources at creation so they will not only be blocked if not compliant but modified to include exportTo: set to “.”.

Let’s apply that with the following command: 


```
kubectl apply -f scope-management/mutation-istio-samens.yaml
```


Now let’s try to create the resources with exportTo: set to “*” (all namespaces):


```
kubectl -n app1 apply -f scope-management/httpbin-ext-tlsorigin-allns.yaml
kubectl -n app1 apply -f scope-management/httpbin-destrule-allns.yaml
```


In this case, even if the resources are not compliant with the constraint defined in the previous section, the resources are created successfully:


```
serviceentry.networking.istio.io/httpbin-ext created
destinationrule.networking.istio.io/httpbin.org created
```


This happens because, even if not compliant, the resources were modified to be compliant with the constraint. 

Let’s look at our ServiceEntry:


```
kubectl -n app1 get serviceentry httpbin-ext -oyaml
```


The final part of the output will be similar to the one below:


```
spec:
  exportTo:
  - .
  hosts:
  - httpbin.org
  location: MESH_EXTERNAL
  ports:
  - name: http
    number: 80
    protocol: HTTP
    targetPort: 443
  - name: https
    number: 443
    protocol: HTTPS
  resolution: DNS
```


The resource has the exportTo: field set to “.” even if the [scope-management/httpbin-ext-tlsorigin-allns.yaml](https://github.com/ggalloro/istio-workshop/blob/main/scope-management/httpbin-ext-tlsorigin-allns.yaml) manifest we applied has its value set to “*”.

The same will happen for the DestinationRule:


```
kubectl -n app1 get destinationrule httpbin.org -oyaml
```


will get you:


```
spec:
  exportTo:
  - .
  host: httpbin.org
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 80
      tls:
        mode: SIMPLE
```


Let’s check our traffic behavior for last time:

Let’s  make a request to httpbin.org from the app1 namespace


```
kubectl -n app1 exec -it "$APP1_POD" -c sleep -- curl http://httpbin.org/anything
```


The url in the request still uses the https scheme, this is expected since app1 is where the ServiceEntry and DestinationRule have been created.

Let’s try the same from the app2 namespace:


```
kubectl -n app2 exec -it "$APP2_POD" -c sleep -- curl http://httpbin.org/anything
```


In that case you will see that the url in the request uses the http scheme, resources are correctly exported only to their same namespace even if at creation they were explicitly set to apply to the whole mesh.

With this option you are not only setting boundaries to your application teams allowing only the creation of traffic management resources in their own namespaces, but also directly modifying the resources to fit in those boundaries.


# Summary

In this article you learned how to set incremental boundaries to your application teams in order to centrally manage the scope of Istio resources:



* You first experienced the default scope that Istio resources have when created
* You learned how it’s possible to set the scope for each resource
* You learned how you can centralize the control of Isto resource scope, with different approaches:
    * Use some Istio global mesh options to change the default scope of resources
    * Use an OPA Gatekeeper constraint to allow, in your Kubernetes cluster, only the creation of Istio resources scoped to their same namespace
    * Add an OPA Gatekeeper mutation to directly modify Istio resources at creation so they are scoped to their same namespace
