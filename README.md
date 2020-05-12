# BucketBindTemplate

## Summary
This is a proposal to enhance the existing Container Object Storage Interface (COSI) specification that focus on the aspect of how buckets get binded to a Pod. We wanted to mimic some of the features of Container Storage Interface (CSI) and allow Application Developers (User's) to have control of how the needed elements to connecting to an Object Storage Backend get injected into the Pod fo his application,If the desired bucket or protocol cannot be satisfied by COSI, the Pod should become unschedulable until this is resolved. 

This proposal focuses on solving this needs by adding a template format into Kubernetes Pod definition that allows developers to specificy their Object Storage needs.

# Motivation
Each application needs different parameters to connect to a certain Object Storage Backend, while some features are standardized such as Access Key and Secret Key for S3-like object storage backends, most existing applications expect those values placed under specific environment variables, we don't want developer to have to go back to change their existing applications in order to adopt COSI, so we propose a way to connect the COSI story to their existing applications without changing the code.

# Goals
 * Define a way for Pods to bind to the `Bucket` or `BucketContent` CRD from the COSI specificiation
 * To allow COSI to be used without changes to the application
 * To simplify secret management and configuration
 
# Vocabulary
 * BucketBindTemplate - a proposed enhacement to the Pod Resource Definition on Kubernetes.
 
 # Proposal
 We propose the addition of a block on Pod definition that would allow developers to define what type of Object Storage they are expecting (S3, GCS, Azure, etc.) and how they are expecting it in their applications, this assumes that the majority of applications are designed to have Object Storage paramters passed via environment variables. We also want to address that some Object Storage backends have SDKs that require files to be present somewhere in the system, for example GCS expect it's `service-account.json` file to be present on a given route.
 
 A developer should be able to specify:
 * The requested `Bucket` by name
 * The protocol it's application works with (S3, GCS, Azure, etc.) 
 * In which enviroment variables the developer is expecting the values for the configuration
 * Configurations and Secrets are managed by `COSI` and injected into the Pod
 
In case his requirements cannot be satisfies, the Pod should become unschedulable which should hint the Adminsitrator to act to satisfy the bucket requirements either by adding the needed `Bucket` or `BucketClass` or by changing the Pod definition to match what the infrastructure is offering.
 
# CRD Enhancement
*BucketBindTemplate*
This is a proposed addition to the Pod CRD to support COSI. A predefined set of variables can be used per protocol for the developer to connect to it's desired environment variables, we also leverage this protocol-specific variables to allow for files to be mounted on specific paths.

Enough information regarding the object storage needs should be exposed for the COSI controller to decide to satisfy or not.

```yaml
bucketBindTemplates:
- metadata:
    name: logs [1]
  spec:
    accessModes: [ "WriteOnly" ] [2]
    bucketClassName: "s3-class" [3]
    env:
      - S3_ENDPOINT: "$(S3_ENDPOINT)" [4]
      - LOGS_BUCKET: "$(S3_BUCKET)" [5]
      - AWS_ACCESS_KEY: "$(S3_ACCESS_KEY)" [6]
      - AWS_SECRET_KEY: "$(S3_SECRET_KEY)" 
      - SOME_OTHER_ENV: "$(S3_ENDPOINT)/$(S3_BUCKET)" [7]
    certs: [8]
      - GCS_SA_JSON: /app/certs/
    # Below are runtime-added values
    bucketContentRef:
        - bucketContent: company-logs [9]
          bucket: logs
```
1. Name of the requested `Bucket`
2. Desired access mode
3. Desired `BucketClass`, here the developer requests a protocol (S3, GCS, Azure, etc.)
4. The developer indicates where the endpoint for the request `S3` protocol should be exposed
5. The actual bucket name might differ from the requested one, here it's exposed to the application via environment variable
6. SDK specific environment variable 
7. More elaborate environment variables are possible due to the templating.
8. Some SDKs expect certificates or files with connection information, the developer can specify the expected mount-point of such files
9. The `BucketContent` the COSI runtime bound to the Pod

### Example Protocol Variables
A set of predefined variables is needed on a per-protocol basis that developers can use to map the desired values. We provide an example for `S3` but we understand other protocols may have additional or different variables.
* **S3_ENDPOINT**: Location of the object storage
* **S3_BUCKET**: Actual name of the bucket
* **S3_REGION**: Bucket region
* **S3_ACCESS_KEY**: Access key 
* **S3_SECRET_KEY**: Secret key
* **S3_HOST_NAME**: The hostname of the endpoint
* **S3_USE_SSL**: Wether or not the endpoint uses SSL


## Example
In this example, we suggest that the application developer is requesting 3 buckets from 2 different providers (S3 and GCS).

The developer is specifying the environment variables for two different buckets on `S3` and expectinig different bucket name results. For `GCS` the developer is requesting that the `service account configuration` be mounted on a given path. 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spark-deployment
  namespace: mapreduce
  labels:
    app: spark
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spark
  template:
    metadata:
      labels:
        app: spark
    spec:
      containers:
        - name: spark
          image: spark:2.4.5
          ports:
            - containerPort: 80
  bucketBindTemplates:
    - metadata:
        name: logs
      spec:
        accessModes: [ "WriteOnly" ]
        bucketClassName: "s3-class"
        env:
          - S3_ENDPOINT: "$(S3_ENDPOINT)"
          - LOGS_BUCKET: "$(S3_BUCKET)"
          - AWS_ACCESS_KEY_ID: "$(S3_ACCESS_KEY)"
          - AWS_SECRET_ACCESS_KEY: "$(S3_SECRET_KEY)"
    - metadata:
        name: bigdata
      spec:
        accessModes: [ "ReadWrite" ]
        bucketClassName: "s3-class"
        env:
          - BIGDATA_BUCKET: "$(S3_BUCKET)"
          - COMPLICATED_ENV_BAR: "https://$(S3_BUCKET).$(S3_HOST_NAME)"
    - metadata:
        name: backups
      spec:
        accessModes: [ "ReadWrite" ]
        bucketClassName: "gcs-class"
        env:
          - BACKUPS_ENDPOINT: "$(GCS_HOST_NAME)"
          - BACKUPS_BUCKET: "$(GCS_CONTAINER)"
          - BACKUPS_ACCESS: "$(GCS_ACCESS_KEY)"
          - BACKUPS_SECRET: "$(GCS_SECRET_KEY)"
        certs:
          - GCS_SA_JSON: /app/certs/
```
