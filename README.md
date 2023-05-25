# Deploy one Spring project to either/both TAP/Kubernetes and Google Cloud Run

## Goals

1. Given a TAP instance, add a new Supply Chain 
1. Build and deploy a Spring project to TAP or GCP
1. Use a single attribute to determine deployment target
1. Use as many common services as possible
1. Use the same Spring source code with no changes

## Prerequistes

- Access to a deployed TAP instance via `tanzu` cli
- Sufficient rights to create Supply Chain objects
- Access to Cloud Run via `gcloud` cli

## Setup

### kubectl config context

Set your workstation's kubectl context to the TAP cluster.

## Executing

Review the two `workload` files in `config`. Note that both are very similar, and point to *the same Spring project*.

### Deploy and test TAP-targeted workload

```
tanzu apps workload create config/workload-tap.yaml
curl or vist url
```

### Deploy and test GCR-targeted workload

```
tanzu apps workload create config workload-gcr.yaml
curl or visit url
```
# Supply Chain Details

https://miro.com/app/board/uXjVMZVbQ8U=/?moveToWidget=3458764550023247935&cot=14

## Selecting a ClusterSupplyChain

TAP/Cartographer hosts one or more `ClusterSupplyChain` objects. Each has a `spec.selectorMatchExpressions` section that, when a new `Workload` shows up, is compared against labels or other attributes. Whichever `ClusterSupplyChain` has the closest match is selected by Cartographer.

The `ClusterSupplyChain` has a `spec.resources` array of objects that comprise the `SupplyChain`. In this case, `ClusterSupplyChain/source-to-url` has a, object spec `resources.image-provider` with a `resources.templateRef` section, made up of a `kind` and `options` to select a concrete implementation.

```
apiVersion: carto.run/v1alpha1
kind: ClusterSupplyChain
metadata:
  annotations: ...
  labels: ...
  name: source-to-url
spec:
  params: ...

  selectorMatchExpressions:              <--- Select closest matching Supply Chain
  - key: apps.tanzu.vmware.com/workload-type
    operator: In
    values:
    - web
    - server
    - worker

  resources:
  ...
  - name: image-provider
    params: ...
    templateRef:
      kind: ClusterImageTemplate
      options:
      - name: kpack-template
        selector:
          matchFields:                    <--- Select closest matching Template
          - key: spec.params[?(@.name=="dockerfile")]
            operator: DoesNotExist
      - name: kaniko-template
        selector:
          matchFields:
          - key: spec.params[?(@.name=="dockerfile")]
            operator: Exists
  ...
  ```
## Selecting a ClusterImageTemplate

The `ClusterImageTemplate` defines what an `Image` should look like and is responsible for getting it built.

--> In Cartographer, a `Template` object gets "stamped out" to the developer namespace to create a concrete implementation of a particular `Workload`'s SupplyChain instance.

The `ClusterImageTemplate` has a `spec.ytt` section that has a section of **Starlark** code (**ytt**'s Python-derived template language) to marshall data variables, followed by a section of Objects that are "stamped out" using the marshalled data variables.

For instance
```
  ytt: |
    #@ load("@ytt:data", "data")

    #@ def merge_labels(fixed_values):            <--- Build the labels...
    #@   labels = {}
    #@   if hasattr(data.values.workload.metadata, "labels"):
    #@     labels.update(data.values.workload.metadata.labels)
    #@   end
    #@   labels.update(fixed_values)
    #@   return labels
    #@ end

...

    apiVersion: kpack.io/v1alpha2
    kind: Image
    metadata:
                                             <--- ...apply the labels.
      labels: #@ merge_labels({ "app.kubernetes.io/component": "build" })
    
```

This `ClusterImageTemplate` stamps out a `kpack.io.Image` object. The --Where does Build come from?

# Extending the ClusterSupply Chain - Options

It seems clear that the way to target different deployment substrates with different dependency or other container-image requirements is to have a different Buildpack create the Image. There seem to be three ways to do this:

1. **Multiple CSC**: Have multiple `ClusterSupplyChain` objects with appropriate `selectorMatchExpression` sections, and add an attribute to the `Workload` to specify the deployment target. This would also come in handy later to select the `Deployment`.
1. **Multiple image-provder.templateRef.options**: Add another `option` in the `spec.resources.[image-provider].templateRef section that looks at some key (q: what does  spec.params[?(@.name=="dockerfile")] refer to specifically?)
1. **Add to the kpack-template to choose a different builder**
1. **Elaborate on Buildpack**: Create a *Buildpack* (or *meta-buildpack*?) with sufficient `detect` functionality to choose the right subsequent buildpack.

## Multiple CSC

Pros:
- Seems easy and keeps things quite separate.
- Easy to think about `Workload` specification and making a concrete `label`
Cons:
- This would fork the overall CSC and not keep together concerns that apply regardless of deployment target. E.g. Spring Boot apps may want to use the same scanners, linters, or other tasks outside the strict build. If an App Operator wanted to modify one of these, they would have to make sure to do it in two places.

## Add a templateRef.option

Pros:
- Can look at attributes that are not just labels. (I suppose the CSC selectorMatchExpression could too?)
- The `ClusterImageTemplate`, especially in the *kpack* implementation, directly maps to the process of choosing a Buildpack. So this is a more "Cartographer" way of choosing a buildpack, and less "Buildpack" way.

Not-sure:
- From a developer perspective, it may seem "magic" how the Image actually gets built. Some magic is good; too much is confusing.

Cons:
- ..

## Extend the kpack-template

This has some promise because if we are going to use Buildpacks, kpack is the tool to do it. In the above option, we could have two `kpack-template`s, like `kpack-template-default` and `kpack-template-gcr`. This sounds promising-- have if the workload specifies nothing, it gets `kpack-template`. If it specifies a deployment target, it gets `kpack-template-[targetname]`. Then developers not aware of the options will get expected behavior. And if someone wants to add a deployment target, they have to create `kpack-template-targetname`. 

## Modify ClusterBuilder

## Elaborate buildpack

Here I think we need to consider the overall lifecycle of Buildpacks and who manages and maintains them. Is it better to have just one uber-Buildpack that handles the concerns of all deployment targets? This means a single resource to maintain, which would be good if there were one person / small team doing it all. But what if the different deployment targets had different SMEs? Might it be better to allow them to work on their specific buildpack in isolation?

Another decision factor is if they are going to do other things outside the platform with the buildpack. For instance, a buildpack can be used directly with Cloud Run. If someone were only working with Cloud Run (or TAP for that matter) it would be negative to also make them concerned with the other deployment targets. There should be artifacts that concern only a single deployment target that are fully effective and do not bring in other concepts.

# Extending the Supply Chain - Decision

This project will implement **2. Multiple image-provder.templateRef.options**. This avoids forking the ClusterSupplyChain, and allows common concerns to be handled within the single Supply Chain. At the same time, it does not tie buildpacks togehter too closely and lets them continue to be fully scoped to their deployment target only.

## Implementation

### Modify ClusterSupplyChain object

Determine appropriate `ClusterSupplyChain.spec.resources.[image-provider].templateRef.options.[...].selector`

Create a new `ClusterImageTemplate` that builds an image appropriate to the deployment target.

# ClusterDelivery

Need to think this through as well.