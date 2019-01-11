# Topsoil Survey data (LUCAS) ingestion

## Data source 
![](doc/jrc.png)

### Description
Physical properties of the surface layer of the soil

### Notes
In 2009, the European Commission extended the periodic Land Use/Land Cover Area Frame Survey (LUCAS) to sample and analyse the main properties of topsoil in 23 Member States of the European Union (EU). This topsoil survey represents the first attempt to build a consistent spatial database of the soil cover across the EU based on standard sampling and analytical procedures, with the analysis of all soil samples being carried out in a single laboratory.

Approximately 20,000 points were selected out of the main LUCAS grid for the collection of soil samples. A standardised sampling procedure was used to collect around 0.5 kg of topsoil (0-20 cm). The samples were dispatched to a central laboratory for physical and chemical analyses.

All samples have been analysed for the percentage of coarse fragments, particle size distribution (% clay, silt and sand content), pH (in CaCl2 and H2O), organic carbon (g/kg), carbonate content (g/kg), phosphorous content (mg/kg), total nitrogen content (g/kg), extractable potassium content (mg/kg) , cation exchange capacity (cmol(+)/kg) and multispectral properties. 

The final database contains 19,967 geo-referenced samples. 

### Links
Methodology, data and results: [https://esdac.jrc.ec.europa.eu/ESDB_Archive/eusoils_docs/other/EUR26102EN.pdf](https://esdac.jrc.ec.europa.eu/ESDB_Archive/eusoils_docs/other/EUR26102EN.pdf)

## Data ingestion

### The microservice pipeline

The ingestion workflow is based on a microservice pipeline developed in python and orchestrated using the Nuclio serverless framework.

These microservices are scalable, highly available and communicate with each other using a Kafka broker/topic enabling the distribution of process workload with buffering and replay capabilities.

The source code of the microservices involved in the pipeline can be found in the ```./pipelines/esdac```directory:

* **00-http-handler**:
expose an API to trigger the ingestion pipeline. It forges a download command from an http POST request and send it to next microservice (01-http-processor).
* **01-http-processor**:
upon receipt of a download command, it downloads and ingests the submitted shapefile archive and sends each record as a GeoJSON feature to the next microservice (02-http-ingestion).
* **02-http-ingestion**:
upon receipt of a GeoJSON feature, a reprojection to the EPSG:4326 CRS is performed before upserting the feature in a MongoDB collection.

### Kafka topics and MongoDB collection

The ```00-boostrap.yaml``` file contains the declarative configuration of two Kafka topics and the ```topsoil``` Mongodb collection with a ```2dsphere``` index on the geometry field.

## Install & deployment

### Docker

The first step is to encapsulate the microservices in Docker containers. The actions of building, tagging and publishing Docker images can be performed using the provided Makefiles.

For each microservice (00-http-handle,01-http-processor and 02-http-ingestion) execute the following bash commands:
1. ```make build``` to build the container image
2. ```make tag``` to tag the newly built container image
3. ```make publish``` to publish the container image on your own Docker repository.

The repository, name and tag of each Docker container image can be overridden using the environment variables DOCKER_REPO, IMAGE_NAME and IMAGE_TAG:
```bash
$> make DOCKER_REPO=index.docker.io IMAGE_NAME=eufast/topsoil-pipe-ingestion IMAGE_TAG=0.2.0 tag
```

### Nuclio / Serverless framework

The Nuclio Serveless framework provides a simple way to describe and deploy microservices (seen as functions) on top of platforms like Kubernetes. In our case, Nuclio handles the subscription to the kafka topics and the execution of the function upon receipt of messages.

A basic configuration is provided and works as is. Don't forget to specify the Docker image created previously. Many configurations are available as the number of replicas, environment variables, resources and triggers.

```yaml
apiVersion: nuclio.io/v1beta1
kind: Function
metadata:
  name: topsoil-pipe-http-trigger
  namespace: fast-platform
  labels:
    platform: fast
    module: core
    data: topsoil
spec:
  alias: latest
  description: Forge a download command from an http POST request
  handler: main:handler
  image: eufast/topsoil-pipe-http-trigger:0.1.0 <-- the container image to set
  replicas: 1
  maxReplicas: 3
  targetCPU: 80
  runtime: python:3.6
  env:
  - name: KAFKA_BOOTSTRAP_SERVER
    value: "kafka-broker.kafka:9092"
  - name: TARGET_TOPIC
    value: topsoil-pipe-download
  resources:
    requests:
      cpu: 10m
      memory: 64Mi
    limits:
      cpu: 1
      memory: 1Gi 
  triggers:
    http:
      attributes:
        ingresses:
          "dev":
            host: api.fast.sobloo.io
            paths:
            - /v1/fast/data/topsoil
      kind: http
      maxWorkers: 5
  version: -1
status:
  state: waitingForResourceConfiguration

```

Please refer to the [official Nuclio documentation](https://nuclio.io/docs/latest/) for more information.

### Kubernetes

To deploy the pipeline on Kubernetes, apply the following YAML manifests:

```bash
$> kubectl create -f 00-bootstrap.yaml
```

```bash
$> cd pipelines/esdac
```

```bash
$> kubectl create -f 00-http-trigger/function.yaml
$> kubectl create -f 01-archive-processor/function.yaml
$> kubectl create -f 02-datastore-ingestion/function.yaml
```

Then check the status of Kubernetes pods:

```bash
$> kubectl -n fast-platform get pod -l module=core,data=topsoil --show-all

NAME                                              READY     STATUS      RESTARTS   AGE
bootstrap-topsoil-data-ntgh7                      0/2       Completed   0          1m
topsoil-pipe-archive-processor-7979dff964-qtt2k   1/1       Running     0          11s
topsoil-pipe-http-trigger-655bc4f5fd-fc8xz        1/1       Running     0          3s
topsoil-pipe-ingestion-77bbb76b7c-x477q           1/1       Running     0          8s
```

### Ingestion

To trigger the ingestion of the data, execute a HTTP POST request as shown below using Postman:

![](doc/postman.png)

