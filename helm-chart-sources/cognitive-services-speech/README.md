# On-premises Speech Service
On-premises Speech（henceforth referred to as **speech-onprem**) Servicve enable customer to build one speech service that is optmized to take advantage of both robust cloud capabilities and edge locality, based on customer's on-premises requirements. This **speech-onprem** service supports two main component services **speech-to-text** and **text-to-speech**, which customer can choose to enable any of them or both.

## Introduction
This chart deploys [Speech Service containers](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/speech-container-howto) on a [Kubernetes](http://kubernetes.io) by [Helm](https://helm.sh).<br/>
This chart supports the deployment of two services: <br/>
* **speech-to-text**
* **text-tot-speech** 

User is able to deploy only one of them or both. <br/>

This chart supports `helm test` command. User is able to apply this command to verify speech service is running properly.
## Prerequisites
- Helm 2.12.3, Kubernetes 1.12.2 
- Helm 2.14.0, Kubernetes 1.14.1

## Resources Required
Please read [Speech Service container computing resource](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/speech-container-howto#the-host-computer) as a reference.

This Helm chart automatically calculates CPU and memory requirements base on how many decodes (concurrent requests) that user specifies and also whether optimization for audio/text input is enabled. <br/>
This Helm chart sets 2 concurrent requests and optimization disabled as default.

|Service| CPU/container | Memory/container|
|---|---|---|
|Speech-to-text| 1 decoder requires minimum 1250 milli cores <br/> If optimization is enabled, requires 2150 milli cores| Request: 4GB<br/> Limited: 8GB|
|Text-to-Speech| 1 concurrency requires minimum 600 milli cores<br> If optimization is enabled, requires 1200 milli cores| Request: 2GB<br/> Limited: 3GB|

## Installing the Chart
1. To deploy Speech Service in kubernetes cluster, two sample files are provided as a reference under dir `speech-onprem/tests`<br/>

   * `containerpreview-sample-deployment.yaml`<br/>
   * `containerpreview-multi-decoders-sample-deployment.yaml`<br/>

    Both of them pull the docker images of Speech Service from **containerpreview.azurecr.io**. To use them, please make sure you have permission to access.
    
    >Note: If you already have docker repo secret setup ready in Kubernetes cluster, skip below steps and go to Step 2 directly.<br/>

    Run this command to login
    ```
    docker login containerpreview.azurecr.io -u <YOUR_USER_NAME> -p <YOUR_PASSWARD>
    ```
    
    In order to make Kubernetes cluster able to pull docker image from **containerpreview.azurecr.io**, we need to transfer the docker credential into cluster.<br/>
    To realize that, please run command
    ```
    kubectl create secret generic containerpreview --from-file=.dockerconfigjson=~/.docker/config.json --type=kubernetes.io/dockerconfigjson
    ```
    `containerpreview` is the secret name created in Kubernetes cluster. That is the name used by `containerpreview-sample-deployment.yaml` and `containerpreview-multi-decoders-sample-deployment.yaml`.<br/>
    `~/.docker/config.json` is the file where docker credential stores. Please use the proper file path on your local environment. <br/>
    
    Run this command to verify
    ```
    kubectl get secrets
    ```
    output
    ```
    NAME                       TYPE                                  DATA      AGE
    containerpreview           kubernetes.io/dockerconfigjson        1         5h
    ...
    ```
2. To install `speech-onprem` helm chart, run
    ```
    helm install <PATH_TO_SPEECH_ONPREM_ON_YOUR_LOCAL> \
             --values <PATH_TO_YOUR_CUSTOMIZED_VALUES_FILE> \
             --name <RELEASE_NAME>
    ``` 
    To apply your own docker repo secret, use `--set ` argument
    ```
    helm install <PATH_TO_SPEECH_ONPREM_ON_YOUR_LOCAL> \
             --set speechToText.image.pullSecrets={<YOUR_OWN_SECRET_NAME>},textToSpeech.image.pullSecrets={YOUR_OWN_SECRET_NAME} \
             --values <PATH_TO_YOUR_CUSTOMIZED_VALUES_FILE> \
             --name <RELEASE_NAME>
    ```
    
    i.e. `speech-onprem` helm chart is located under Windows D: drive on my local. Commands below access it via WindowsLinuxSubsystem (WSL). <br/>
    ```
    helm install /d/kubernetes/helm/speech-onprem \
            --values /d/kubernetes/helm/speech-onprem/test/containerpreview-sample-deployment.yaml \
            --name onprem-sample
    ```
    ```
    helm install /d/kubernetes/helm/speech-onprem \
            --set speechToText.image.pullSecrets={my-container-preview},textToSpeech.image.pullSecrets={my-container-preview} \
            --values /d/kubernetes/helm/speech-onprem/test/containerpreview-sample-deployment.yaml \
            --name onprem-sample
    ```
 
3. This chart supports `helm test` command. It creates `speech-to-text-readiness-test` and `text-to-speech-readiness-test` pods in the cluster to verify `speech-to-text` and `text-to-speech` services are running successfully.
    ```
    helm test <RELEASE_NAME>
    ```
    i.e.
    ```
    helm test onprem-sample
    ```
   
## Configuration
The Helm chart ships with reasonable defaults. However, since it is named on-premises, it should definitely support customization and configuration overrides.<br/>
To apply customized configurations, 
1. override helm values though command: <br/>
[`helm install --set <KEY>=<VALUE> ...`](https://helm.sh/docs/helm/#options-21)
2. create a new separate **values.yaml** file and apply it through command:<br/>
[`helm install --values <YOUR_VALUES_FILE_PATH>`](https://helm.sh/docs/helm/#options-21)

Please check below sections for the details of configurable options of the Helm chart.

> **Note**: configurable options may change in future.

### SPEECH-ONPREM (umbrella chart)
> Values in umbrella chart override the corresponding sub-chart values.<br/>
> Therefore, if user chooses .yaml file to apply on-premises customized values, it should follow one of the methods below:<br/>
> 1. added in umbrella chart values.yaml file. <br/>
> 2. create a new separate .yaml file to override. <br/>
>
> No customized values in sub-chart values.yaml, <br/>
> No customized values in sub-chart values.yaml, <br/>
> No customized values in sub-chart values.yaml. <br/>

|Parameter|Description|Values|Default|
| --- | --- | --- | --- |
|`speechToText.enabled`|Specifies whether enable **speech-to-text** service| true/false| `true` |
|`speechToText.verification.enabled`| Specifies whether enable `helm test` capability for **speech-to-text** service | true/false | `true` |
|`speechToText.verification.image.registry`| Specifies docker image repository that `helm test` uses to test **speech-to-text** service. Helm creates separate pod inside the cluster for testing and pulls the test-use image from this registry| valid docker registry | `docker.io`|
|`speechToText.verification.image.repository`| Specifies docker image repository that `helm test` uses to test **speech-to-text** service. Helm test pod uses this repository to pull test-use image| valid docker image repository |`antsu/on-prem-client`|
|`speechToText.verification.image.tag`| Specifies docker image tag that used `helm test` for **speech-to-text** service. Helm test pod uses this tag to pull test-use image | valid docker image tag | `latest`|
|`speechToText.verification.image.pullByHash`| Specifies whether test-use docker image is pulled by hash.<br/> If `yes`, `speechToText.verification.image.hash` should be added, with valid image hash value. <br/> It's `false` by default.|true/false| `false`|
|`speechToText.verification.image.arguments`| Specifies the arguments to execute test-use docker image. Helm test pod passes these arguments to container when running `helm test`| valid arguments as the test docker image requires |`"./speech-to-text-client"`<br/> `"./audio/whatstheweatherlike.wav"` <br/> `"--expect=What's the weather like"`<br/>`"--host=$(SPEECH_TO_TEXT_HOST)"`<br/>`"--port=$(SPEECH_TO_TEXT_PORT)"`|
|`textToSpeech.enabled`|Specifies whether enable **text-to-speech** service| true/false| `true` |
|`textToSpeech.verification.enabled`| Specifies whether enable `helm test` capability for **text-to-speech** service | true/false | `true` |
|`textToSpeech.verification.image.registry`| Specifies docker image repository that `helm test` uses to test **text-to-speech** service. Helm creates separate pod inside the cluster for testing and pulls the test-use image from this registry| valid docker registry | `docker.io`|
|`textToSpeech.verification.image.repository`| Specifies docker image repository that `helm test` uses to test **text-to-speech** service. Helm test pod uses this repository to pull test-use image| valid docker image repository |*`antsu/on-prem-client`*|
|`textToSpeech.verification.image.tag`| Specifies docker image tag that used `helm test` for **text-to-speech** service. Helm test pod uses this tag to pull test-use image | valid docker image tag | `latest`|
|`textToSpeech.verification.image.pullByHash`| Specifies whether test-use docker image is pulled by hash.<br/> If `yes`, `textToSpeech.verification.image.hash` should be added, with valid image hash value. <br/> It's `false` by default.|true/false| `false`|
|`textToSpeech.verification.image.arguments`| Specifies the arguments to execute test-use docker image. Helm test pod passes these arguments to container when running `helm test`| valid arguments as the test docker image requires |`"./text-to-speech-client"`<br/> `"--input='What's the weather like'"` <br/> `"--host=$(TEXT_TO_SPEECH_HOST)"`<br/>`"--port=$(TEXT_TO_SPEECH_PORT)"`|

### SPEECH-TO-TEXT (subchart: charts/speechToText)
> Again, no customized values in sub-chart values.yaml! <br/>
> 
> Add prefix `speechToText.` on any parameter below into your own .yaml file/umbrella chart's values.yaml can override the corresponding values here. <br/>
> i.e. `speechToText.numberOfConcurrentRequest` overrides `numberOfConcurrentRequest`.<br/>

|Parameter|Description|Values|Default|
| --- | --- | --- | --- |
|`enabled`| Specifies whether enable **speech-to-text** service| true/false| `false`|
|`numberOfConcurrentRequest`| Specifies how many concurrent requests for **speech-to-text** service.<br/> This chart automatically calculate CPU and memory resources, based on this value.| int | `2` |
|`optimizeForAudioFile`| Specifies if service needs to optimize for audio input via audio file. <br/> If `yes`, this chart will allocate more CPU resource to service. <br/> Default is `false`| true/false |`false`|
|`image.registry`| Specifies the **speech-to-text** docker image registry| valid docker image registry| |
|`image.repository`| Specifies the **speech-to-text** docker image repository| valida docker image repository| |
|`image.tag`| Specifies the **speech-to-text** docker image tag| valid docker image tag| |
|`image.pullSecrets`| Specifies the image secrets for pulling **speech-to-text** docker image| valida secrets name| |
|`image.pullByHash`| Specifies if pulling docker image by hash.<br/> If `yes`, `image.hash` is required to have as well.<br/> If `no`, set it as 'false'. Default is `false`.| true/false| `false`|
|`image.hash`| Specifies **speech-to-text** docker image hash. Only use it when `image.pullByHash:true`.| valid docker image hash | |
|`image.args.eula`| One of the required arguments by **speech-to-text** container, which indicates you've accepted the license.<br/> The value of this option must be: accept| `accept`, if you want to use the container | |
|`image.args.billing`| One of the required arguments by **speech-to-text** container, which specifies the billing endpoint URI<br/> The billing endpoint URI value is available on the Azure portal's Speech Overview page.|valid billing endpoint URI||
|`image.args.apikey`| One of the required arguments by **speech-to-text** container, which is used to track billing information.| valid apikey||
|`service.type`| Specifies the type of **speech-to-text** service in Kubernetes. <br/> [Kubernetes Service Types Instruction](https://kubernetes.io/docs/concepts/services-networking/service/)<br/> Default is `LoadBalancer` (please make sure you cloud provider supports) | valid Kuberntes service type | `LoadBalancer`|
|`service.port`| Specifies the port of **speech-to-text** service| int| `80`|
|`service.annotations`| The annotations user can add to **speech-to-text** service metadata. For instance:<br/> **annotations:**<br/>`   ` **some/annotation1: value1**<br/>`  ` **some/annotation2: value2** | annotations, one per each line| |
|`serivce.autoScaler.enabled`| Specifies if enable [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)<br/> If enabled, `speech-to-text-autoscaler` will be deployed in the Kubernetes cluster | true/false| `true`|
|`service.podDisruption.enabled`| Specifies if enable [Pod Disruption Budget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)<br/> If enabled, `speech-to-text-poddisruptionbudget` will be deployed in the Kubernetes cluster| true/false| `true`|

### TEXT-TO-SPEECH (subchart: charts/textToSpeech)
> Again, no customized values in sub-chart values.yaml! <br/>
> 
> Add prefix `textToSpeech.` on any parameter below into your own .yaml file/umbrella chart's values.yaml can override the corresponding values here. <br/>
> i.e. `textToSpeech.numberOfConcurrentRequest` overrides `numberOfConcurrentRequest`.<br/>

|Parameter|Description|Values|Default|
| --- | --- | --- | --- |
|`enabled`| Specifies whether enable **text-to-speech** service| true/false| `false`|
|`numberOfConcurrentRequest`| Specifies how many concurrent requests for **text-to-speech** service.<br/> This chart automatically calculate CPU and memory resources, based on this value.| int | `2` |
|`optimizeForTurboMode`| Specifies if service needs to optimize for high usage. <br/> If `yes`, this chart will allocate more CPU resource to service. <br/> Default is `false`| true/false |`false`|
|`image.registry`| Specifies the **text-to-speech** docker image registry| valid docker image registry| |
|`image.repository`| Specifies the **text-to-speech** docker image repository| valida docker image repository| `|
|`image.tag`| Specifies the **text-to-speech** docker image tag| valid docker image tag| |
|`image.pullSecrets`| Specifies the image secrets for pulling **text-to-speech** docker image| valida secrets name||
|`image.pullByHash`| Specifies if pulling docker image by hash.<br/> If `yes`, `image.hash` is required to have as well.<br/> If `no`, set it as 'false'. Default is `false`.| true/false| `false`|
|`image.hash`| Specifies **text-to-speech** docker image hash. Only use it when `image.pullByHash:true`.| valid docker image hash | |
|`image.args.eula`| One of the required arguments by **text-to-speecht** container, which indicates you've accepted the license.<br/> The value of this option must be: accept| `accept`, if you want to use the container | |
|`image.args.billing`| One of the required arguments by **text-to-speech** container, which specifies the billling endpoint URI<br/> The billing endpoint URI value is available on the Azure portal's Speech Overview page.|valid billing endpoint URI||
|`image.args.apikey`| One of the required arguments by **text-to-speech** container, which is used to track billing information.| valid apikey||
|`service.type`| Specifies the type of **text-to-speech** serivce in Kubernetes. <br/> [Kubernetes Service Types Instruction](https://kubernetes.io/docs/concepts/services-networking/service/)<br/> Default is `LoadBalancer` (please make sure you cloud provider supports) | valid Kuberntes service type | `LoadBalancer`|
|`service.port`| Specifies the port of **text-to-speech** service| int| `80`|
|`service.annotations`| The annotations user can add to **text-to-speech** service metadata. For instance:<br/> **annotations:**<br/>`   ` **some/annotation1: value1**<br/>`  ` **some/annotation2: value2** | annotations, one per each line| |
|`serivce.autoScaler.enabled`| Specifies if enable [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)<br/> If enabled, `text-to-speech-autoscaler` will be deployed in the Kubernetes cluster | true/false| `true`|
|`service.podDisruption.enabled`| Specifies if enable [Pod Disruption Budget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)<br/> If enabled, `text-to-speech-poddisruptionbudget` will be deployed in the Kubernetes cluster| true/false| `true`|

## Helm Test
A test-purpose docker image has been created and is available at [docker.io/antsu/on-prem-client:latest](https://hub.docker.com/r/antsu/on-prem-client). <br/>
User can create your own customized test docker image based (or not) on it to add more testing features if you want.<br/>
<br/>
To modify the existing `helm test` feature inside helm chart, or to apply your own customized test docker image, follow the `speechToText.verification` and `textToSpeech.verification` sections in umbrella chart's `values.yaml` as a sample. Those values are open to replace by yours. <br/>
<br/>
To get Helm tests run, please read [Helm Test Command Instruction](https://helm.sh/docs/helm/#helm-test). Basically, the command should be:<br/>
```
helm test <RELEASE_NAME>
```

## Uninstalling the Chart
To uninstall or delete `speech-onprem` completely, run:<br/>
```
helm del --purge <RELEASE_NAME>
``` 