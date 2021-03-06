---
layout: post
title: Early Access Kubernetes V2 Provider Guide
order: 100
# No redirect - setup for ec2 vs. halyard installer is diffferent.
---
{% include components/legacy_documentation.html %}

This guide includes:
{:.no_toc}
* This is a placeholder for an unordered list that will be replaced with ToC. To exclude a header, add {:.no_toc} after it.
{:toc}


## Kubernetes V2 Provider Overview
The new Kubernetes provider is centered on delivering and promoting Kubernetes manifests to different Kubernetes environments. These manifests are delivered to Spinnaker using [artifacts](https://www.spinnaker.io/reference/artifacts/in-kubernetes-v2/#kubernetes-objects-as-artifacts) and applied to the target cluster using `kubectl` in the background. Currently, there is support to supply artifacts through Git, GCS and Google Pub/Sub.  The V2 provider manages the version of the artifacts deployed to Kubernetes by appending a string such as `v421` to the resource that was deployed to Kubernetes.  This allows for easy rollbacks and management of resources.

### What To Expect From The V2 Provider
The new Kubernetes provider is very much a work in progress and is subject to changes in the UI and APIs. The user experience is still a work in progress and we're interested in understanding real-world usage patterns as we iterate on the provider to optimize the experience.  

### Current Limitations
-  The only supported services for artifact delivery are Github, GCS, or Google Pub/Sub. S3, SQS, and SNS are currently not supported.
- Native Spinnaker deployment strategies such as red/black, highlander, rolling red/black deployment are not supported. If these strategies are desired consider using the deployment object in your manifests.
- While you're able to deploy all Kubernetes resource types, the V2 provider only considers `containers` and `configMaps` for [binding to the deployed manifest](https://www.spinnaker.io/reference/artifacts/in-kubernetes-v2/#kubernetes-objects-as-artifacts). `secrets` and other resource types are coming soon.    
- You cannot manually trigger the pipeline, you have to use Github triggers.

### Available Manifest Based Stages

There are 4 stages that are available:

1. **Deploy Manifest** -  Uses `kubectl apply` to deploy a manifest.  Spinnaker will wait until the manifest stabilizes in the Kubernetes cluster or reach the expected capacity on healthy pods.

2. **Delete Manifests** - Removes the manifests and their corresponding artifacts in kubernetes based on different types and labels.  

3. **Scale Manifests** - Scales replica sets to a given static size.

4. **Rollback Manifest** - Allows rollback of a given Kubernetes artifact by a given number of versions.  Helpful in cases of a failed deployment or failed manual QA.

## Setting Up The V2 Provider

Setting up the V2 provider is similar to the [V1 Kubernetes configuration](http://docs.armory.io/admin-guides/configure_kubernetes/#configure-clouddriver-to-use-the-kubectl-config-file) however we'll need to change the provider flag in `/opt/spinnaker/config/clouddriver-local.yml` to `v2`.  For example:

```
kubernetes:
  enabled: true
  accounts:
    - name: k8s-v2
      providerVersion: v2
      kubeconfigFile: /opt/spinnaker/credentials/kubeconfig
```

We'll also need to enable artifact handling in Spinnaker by setting a flag in `/opt/spinnaker/config/spinnaker-local.yml`

```
features:
  artifacts:
    enabled: true
```

And we'll need to configure the Github artifact account in `/opt/spinnaker/config/clouddriver-local.yml`

```yaml
artifacts:
  github:
    enabled: true
    accounts:
    - name: github
      username: BOT_USERNAME
      token: YOURTOKEN
```      

Then restart Armory Spinnaker: `service armory-spinnaker restart`

## Creating a Kubernetes V2 Pipeline

### Configuring The Pipeline Trigger
We'll begin by creating a pipeline that is triggered from Kubernetes artifacts and delivered through Github.  Below we'll define two artifacts that will be deployed as Kubernetes manifests: `deployment.yml` and `config-map.yml` which are valid Kubernetes manifests.  Make sure to select the source as `Github`.

![artifacts](/images/page.png)

After configuring the artifacts we'll need to associate them with a Github trigger so the pipeline is triggered whenever there are modifications pushed to Github.  For example, the pipeline below will only trigger when _either_ `deployment.yml` _or_ `config-map.yml` are modified and pushed to the repository.  If the manifest isn't modified it'll use the latest version that was deployed.

> Note: If you haven't already, you'll need to configure [Github webhooks](https://www.spinnaker.io/setup/features/notifications/#github) for your Spinnaker instance.

![github trigger](/images/trigger.png)


### Configuring The Config Map Manifest Delivery

We'll configure the `configMap` to be deployed first. Add a `Deploy (Manifest)` stage to your pipelines.

![deploy manifest](/images/deploy_manifest.png)


Once you've added the stage, select `Artifact` from the `Manifest Source` below and it will allow you to choose one of the expected artifacts that we configured in the previous section.  Choose `config-map.yml` and hit `save`. Spinnaker will deploy the chosen artifact but append a version to the name of the artifact. For [our example config map](https://github.com/Armory/spinnaker-k8s-v2-example/blob/master/config-map.yml). So for the name `k8-v2-config-map` it will appear in the Kubernetes cluster with `k8-v2-config-map-v001`.

![config map](/images/config-map.png)

### Configuring Deployment Manifest Delivery

Next we'll configure a new `Deploy (Manifest)` stage to deploy the [deployment.yml](https://github.com/Armory/spinnaker-k8s-v2-example/blob/master/deployment.yml) manifest.  This manifest references our config-map as a volume and it's source will be replaced by the versioned artifact deployed in the previous step: `k8-v2-config-map-v001`.  So if our `deployment.yml` contains the following:

```yaml
volumes:
  - name: k8-config
    configMap:
      name: k8-v2-config-map
```

the name will be replaced with the properly versioned artifact:
```
volumes:
  - name: k8-config
    configMap:
      name: k8-v2-config-map-v000
```

![deployment](/images/Image 2018-01-26 at 5.36.53 PM.png)

### Executing The Pipeline

Your final pipeline should look similar to the one below.
![final pipeline](/images/pipeline.png)

In order to execute your pipeline the first time you'll need to edit both the `config-map.yml` and `deployment.yml`, commit the changes to git and push the changes to Github. The pipeline should trigger and execute flawlessly =)

but if it doesn't, don't hesitate to reach out: http://go.armory.io/chat
