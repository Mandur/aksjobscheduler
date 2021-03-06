# Scheduling Jobs in AKS with Virtual Kubelet

![Scheduling Jobs in AKS with Virtual Kubelet](./media/aksjobscheduler.png)

Most Kubernetes sample applications demonstrate how to run web applications or databases in the orchestrator. Those workloads run until terminated.

However, Kubernetes is also able to run jobs, or in other words, a container that runs for a finite amount of time.

Jobs in Kubernetes can be classified in two:

- [Cron Job](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/): run periodically based on defined schedule
- [Run to Completion](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/): run N amount of times

This repository contains a demo application that schedules `Run to Completion` jobs in a Kubernetes cluster taking advantage of Virtual Kubelet in order to preserved allocated resources. It has an option to integration with [Event Grid](https://docs.microsoft.com/en-us/azure/event-grid/overview) to notify once a job has been completed.

## Scenario

Company is using a Kubernetes cluster to host a few applications. Cluster resource utilisation is high and team wants to avoid oversizing. One of the teams has new requirement to run jobs with unpredictable loads. In high peaks, the required compute resources will exceed what is available in the cluster.

A solution to this problem is to leverage Virtual Kubelet, scheduling jobs outside the cluster in case workload would starve available resources.

In my experience running the sample application using Virtual Kubelet had the following results:

|Location|Image Repository|Min Duration|Max Duration|
|-|-|-|-|
|Cluster|doesn't matter (image caching)|7 secs|35 secs|
|ACI|ACR|35 secs|60 secs|
|ACI|Docker Hub|35 secs| 80 secs|

**Disclaimer**\
There are other options to run jobs in Azure (i.e. Azure Functions, Azure Batch, WebJobs). The option presented here might suit better a team that wishes to leverage containers and orchestrators experience.

## Running containers in Virtual Kubelet in Azure

[Virtual Kubelet](https://github.com/virtual-kubelet/virtual-kubelet) is an open source project allowing Kubernetes to schedule workloads outside of the reserved cluster nodes.

For more information refer to documentation:

- [Azure Container Instance](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-overview)
- [Use Virtual Kubelet in Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/virtual-kubelet)

### Virtual Kubelet pulling private images from ACR

According to the documentation it should be possible to set a service principal with access to ACR. However, it [does not seem to work](https://github.com/virtual-kubelet/virtual-kubelet/issues/192).
You will see the following message on the job pod:

```text
Status:             Pending
Reason:             ProviderFailed
Message:            api call to https://management.azure.com/subscriptions/xxxxxx/resourceGroups/xxxxx/providers/Microsoft.ContainerInstance/containerGroups/xxxxxx?api-version=2018-09-01: got HTTP response status code 400 error code "InaccessibleImage": The image 'xxxxx.azurecr.io/aksjobscheduler-worker-dotnet:1.0' in container group 'xxxx' is not accessible. Please check the image and registry credential.
```

The workaround describe in the GitHub issue requires us to create a secret and pass it to the [pullImageSecrets](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
Creating the secret is documented [here](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-aks#access-with-kubernetes-secret):

```bash
$ kubectl create secret docker-registry my-acr-auth --docker-server xxx.azurecr.io --docker-username "<service principal id>" --docker-password "<service principal password>" --docker-email "<email address>"
```

Additionally, deploy the Jobs API with the correct `JOBIMAGEPULLSECRET`

## Running jobs in Azure Container Instance

Running a job in Kubernetes is similar to deploying a web application. The difference is how we define the YAML file.

Take for example the [Kubernetes demo job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#running-an-example-job) defined by the YAML below:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

Running the job in Kubernetes with kubectl is demonstrated below:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/controllers/job.yaml
```

Viewing running jobs

```bash
$ kubectl get jobs
NAME           DESIRED   SUCCESSFUL   AGE
pi             1         1            15s
```

Viewing the job result:

```bash
# find first pod where the job name is "pi", then list the logs
POD_NAME=$(kubectl get pods -l "job-name=pi" -o jsonpath='{.items[0].metadata.name}') && clear && kubectl logs $POD_NAME

3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385 REMOVED-TO-SAVE-SPACE
```

To run the same job using Virtual Kubelet add the pod selection/affinity information as the yaml below:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: aci-pi
spec:
  template:
    spec:
      containers:
      - name: aci-pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
        resources:
          requests:
            cpu: "1"
            memory: "1Gi"
      restartPolicy: Never
      nodeSelector:
        beta.kubernetes.io/os: linux
        kubernetes.io/role: agent
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Equal
        value: azure
        effect: NoSchedule
  backoffLimit: 4
```

## Sample application

The code found in this repository has 2 components:

- **Jobs API (/server)**\
Web API to manage jobs. Executing of new jobs is redirect depending on the thresholds defined for local cluster workloads.\
The implementation uses the Kubernetes API to schedule jobs. State is obtained from native kubernetes objects and metadata.\
Job input file is copied to Azure Storage. Index files are created to enable parallelism in job execution.

- **Job Worker (/workers/dotnet)**\
Sample implementation of job worker.

Starting a new job requires sending a file through the Jobs API. A post request to http://api-url/jobs with the input file (as upload file like the example below) creates a job:

```json
{ "id": "1", "value1": 3123, "value2": 321311, "op": "+" }
{ "id": "2", "value1": 3123, "value2": 321311, "op": "-" }
{ "id": "3", "value1": 3123, "value2": 321311, "op": "/" }
{ "id": "4", "value1": 3123, "value2": 321311, "op": "*" }
```

The result of the create job request is the jobID. The outcome of a job performed by workers is the following:

```json
{"id":"1","value1":3123.0,"value2":321311.0,"op":"+","result":324434.0}
{"id":"2","value1":3123.0,"value2":321311.0,"op":"-","result":-318188.0}
{"id":"3","value1":3123.0,"value2":321311.0,"op":"/","result":0.009719555197301057}
{"id":"4","value1":3123.0,"value2":321311.0,"op":"*","result":1003454253.0}
```

As mentioned before, the Jobs API will create index files that will identity where each N amount of lines start on the input file. Those small index files will be used by workers to [lease](https://docs.microsoft.com/en-us/rest/api/storageservices/lease-blob) a range of lines to be processed, preventing parallel workers from processing the same content.

## Running sample application in AKS

1. Create a new storage account (the sample application will store input and output files there)

2. Deploy application to your AKS with the provided yaml file. Keep in mind that the deployment will create a new public IP since the service is of type LoadBalancer. Modify the deployment yaml file to contain your storage credentials.

If the target Kubernetes cluster has role based access control (RBAC), we need to give access permission to the batch job api. The file `deployment-rbac` will create the service account, role and role binding needed before creating the deployment and service:

```bash
# with rbac (service account assignment on pod)
kubectl apply -f deployment-rbac.yaml

# without rbac
kubectl apply -f deployment.yaml
```

3. Watch the Jobs API logs

```bash
POD_NAME=$(kubectl get pods -l "app=jobscheduler" -o jsonpath='{.items[0].metadata.name}') && clear && kubectl logs $POD_NAME -f
```

4. Create a new job by uploading an input file (modify the file path)

```bash
curl -X POST \
  http://{location-of-jobs-api}/jobs \
  -H 'Content-Type: multipart/form-data' \
  -H 'cache-control: no-cache' \
  -H 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' \
  -F file=@/path/to/sample_large_work.json
```

5. Look at the job status

```bash
curl http://{location-of-jobs-api}/jobs/{job-id}

{"id":"2018-10-4610526630846599105","status":"Complete","startTime":"2018-10-31T12:31:52Z","completionTime":"2018-10-31T12:33:57Z","succeeded":13,"parallelism":4,"parts":13,"completions":13,"storageContainer":"jobs","storageBlobPrefix":"2018-10/4610526630846599105","runningOnAci":true}
```

6. Download the job result once finished in parts or as a single file (remove the `part` query string parameter)

```bash
curl http://{location-of-jobs-api}/jobs/{job-id}/results?part=13 --output results-part-13.json
```

### Jobs Web API

Besides interacting with Azure Storage, the API schedules jobs in Kubernetes using the client API ([using the go-client](https://github.com/kubernetes/client-go)).

|Action|Description|
|-|-|
|POST /jobs|Receives a file upload, copy to az storage, create index files and starts K8s job|
|GET /jobs|Retrieves all jobs|
|GET /jobs/{id}|Retrieves detail of a job|
|GET /jobs/{id}/results|Retrieves result of a complete job. Use the query string `part` to download small parts|

The job creation process is the following:

```text
While (reads line from input file)
  Add line to local buffer
  Whenever N lines were read, create a index file
  Whenever local buffer is almost full (~4 MB), write to blob
End While

Create Kubernetes job based on total_lines. Data returned from the Jobs API is retrieved from the Azure Storage or Kubernetes objects & metadata.
```

#### Jobs API configuration

The API can be configures using environment variables as detailed here:

|Environment Variable|Description|Default value|
|-|-|-|
|STORAGEACCOUNT|Storage account name||
|STORAGEKEY|Storage account key||
|CONTAINERNAME|Storage container name|jobs|
|MAXPARALLELISM|Max parallelism for local cluster|2|
|ACIMAXPARALLELISM|Max parallelism for ACI|50|
|LINESPERJOB|Lines per job|100'000|
|ACICOMPLETIONSTRIGGER|Defines the amount of completions necessary to execute the job using virtual kubelet|Default is 6. 0 to disable ACI|
|ACIHOSTNAME|Defines the ACI host name|virtual-kubelet-virtual-kubelet-linux-westeurope|
|JOBIMAGE|Image to be used in job|fbeltrao/aksjobscheduler-worker-dotnet:1.0|
|JOBIMAGEPULLSECRET|Defines the image pull secret when using a private image repository. Secret created from the command `kubectl create secret docker-registry ...`||
|JOBCPULIMIT|Job CPU limit for local cluster|0.5|
|ACIJOBCPULIMIT|Job CPU limit for ACI|1|
|JOBMEMORYLIMIT|Job Memory limit for local cluster|256Mi|
|ACIJOBMEMORYLIMIT|Job Memory limit for ACI|1Gi|
|EVENTGRIDENDPOINT|Event Grid event endpoint to publish when job is done||
|EVENTGRIDSASKEY|Event Grid sas key publish when job is done||
|DEBUGLOG|Enabled debug log level|false|

Source code is located at /server

### Worker

The provided worker implementation is based on .NET Core. A worker execution happens in the following way:

```text
If could acquire lease to one of the index files
  Periodically renew lease
  Process lines defined by leased index file, writing to output file
  Delete leased index file
End If
```

Source code is located at /workers/dotnet/

### Event grid integration

There are two ways to identify when a job has finished:

- By polling the API /jobs/{id}

```json
{
    "id": "2018-10-8405256245329624086",
    "status": "Complete",
    "startTime": "2018-10-30T15:43:10Z",
    "completionTime": "2018-10-30T15:43:22Z",
    "succeeded": 2,
    "parallelism": 1,
    "completions": 2,
    "storageContainer": "jobs",
    "storageBlobPrefix": "2018-10/8405256245329624086",
    "runningOnAci": false
}
```

- By using Event Grid. If an event grid endpoint and sas keys was provided to the Jobs API an additional completion will be scheduled. The worker will send an event grid notification if no index files is found (all indexes were completed).

## Improvements

This is a list of things that should be improved:

- Accessing ACR images from virtual kubelet ([GitHub issue](https://github.com/virtual-kubelet/virtual-kubelet/issues/192))
- Worker job is responsible for sending event grid notifications. Should be the responsibility of the scheduler
- Worker job should be able to process multiple files to decrease the time needed to create new pods
- Review io reading from storage in job scheduler. I am new to golang, there might be better ways to implement it
- Use proper dependency management in go project