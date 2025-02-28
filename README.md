![sidekick](https://raw.githubusercontent.com/minio/sidekick/master/sidekick_logo.png)

![build](https://github.com/minio/sidekick/workflows/Go/badge.svg) ![license](https://img.shields.io/badge/license-AGPL%20V3-blue)

![GitHub Downloads][gh-downloads]

*sidekick* is a high-performance sidecar load-balancer. By attaching a tiny load balancer as a sidecar to each of the client application processes, you can eliminate the centralized loadbalancer bottleneck and DNS failover management. *sidekick* automatically avoids sending traffic to the failed servers by checking their health via the readiness API and HTTP error returns.

# Architecture
![architecture](https://raw.githubusercontent.com/minio/sidekick/master/arch_sidekick.png)

**Demo** ![sidekick-demo](https://raw.githubusercontent.com/minio/sidekick/master/sidekick-demo.gif)

# Install

## Binary Releases

| OS      | ARCH    | Binary                                                                                       |
|:-------:|:-------:|:--------------------------------------------------------------------------------------------:|
| Linux   | amd64   | [linux-amd64](https://github.com/minio/sidekick/releases/latest/download/sidekick-linux-amd64)         |
| Linux   | arm64   | [linux-arm64](https://github.com/minio/sidekick/releases/latest/download/sidekick-linux-arm64)         |
| Linux   | ppc64le | [linux-ppc64le](https://github.com/minio/sidekick/releases/latest/download/sidekick-linux-ppc64le)     |
| Linux   | s390x   | [linux-s390x](https://github.com/minio/sidekick/releases/latest/download/sidekick-linux-s390x)         |
| Apple   | amd64   | [darwin-amd64](https://github.com/minio/sidekick/releases/latest/download/sidekick-darwin-amd64)       |
| Windows | amd64   | [windows-amd64](https://github.com/minio/sidekick/releases/latest/download/sidekick-windows-amd64.exe) |

You can also verify the binary with [minisign](https://jedisct1.github.io/minisign/) by downloading the corresponding [`.minisig`](https://github.com/minio/sidekick/releases/latest) signature file. Then run:
```
minisign -Vm sidekick-<OS>-<ARCH> -P RWTx5Zr1tiHQLwG9keckT0c45M3AGeHD6IvimQHpyRywVWGbP1aVSGav
```

## Docker

Pull the latest release via:
```
docker pull minio/sidekick:v2.0.0
```

## Build from source

```
go install -v github.com/minio/sidekick@latest
```

> You will need a working Go environment. Therefore, please follow [How to install Go](https://golang.org/doc/install).
> Minimum version required is go1.17

# Usage

```
NAME:
  sidekick - High-Performance sidecar load-balancer

USAGE:
  sidekick - [FLAGS] SITE1 [SITE2..]

FLAGS:
  --address value, -a value           listening address for sidekick (default: ":8080")
  --health-path value, -p value       health check path
  --read-health-path value, -r value  health check path for read access - valid only for failover site
  --health-port value                 health check port (default: 0)
  --health-duration value, -d value   health check duration in seconds (default: 5s)
  --insecure, -i                      disable TLS certificate verification
  --log, -l                           enable logging
  --trace value, -t value             enable request tracing - valid values are [all,application,minio] (default: "all")
  --quiet, -q                         disable console messages
  --json                              output sidekick logs and trace in json format
  --debug                             output verbose trace
  --cacert value                      CA certificate to verify peer against
  --client-cert value                 client certificate file
  --client-key value                  client private key file
  --cert value                        server certificate file
  --key value                         server private key file
  --help, -h                          show help
  --version, -v                       print the version
```

## Examples

### Load balance across a web service using DNS provided IPs
```
$ sidekick --health-path=/ready http://myapp.myorg.dom
```

### Load balance across 4 MinIO Servers (http://minio1:9000 to http://minio4:9000)
```
$ sidekick --health-path=/minio/health/ready --address :8000 http://minio{1...4}:9000
```

### Two sites with 4 servers each
```
$ sidekick --health-path=/minio/health/ready http://site1-minio{1...4}:9000 http://site2-minio{1...4}:9000
```

## Realworld Example with spark-operator

As spark *driver*, *executor* sidecars, to begin with install spark-operator and MinIO on your kubernetes cluster

**optional** create a kubernetes namespace `spark-operator`
```
kubectl create ns spark-operator
```

### Configure *spark-operator*

We shall be using maintained spark operator by GCP at https://github.com/GoogleCloudPlatform/spark-on-k8s-operator

```
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
helm install spark-operator incubator/sparkoperator --namespace spark-operator  --set sparkJobNamespace=spark-operator --set enableWebhook=true
```

### Install *MinIO*
```
helm install minio-distributed stable/minio --namespace spark-operator --set accessKey=minio,secretKey=minio123,persistence.enabled=false,mode=distributed
```

> NOTE: persistence is disabled here for testing, make sure you are using persistence with PVs for production workload.
> For more details read our helm [documentation](https://github.com/helm/charts/tree/master/stable/minio)

Once minio-distributed is up and running configure `mc` and upload some data, we shall choose `mybucket` as our bucketname.

Port-forward to access minio-cluster locally.
```
kubectl port-forward pod/minio-distributed-0 9000
```

Create bucket named `mybucket` and upload some text data for spark word count sample.
```
mc config host add minio-distributed http://localhost:9000 minio minio123
mc mb minio-distributed/mybucket
mc cp /etc/hosts minio-distributed/mybucket/mydata/{1..4}.txt
```

### Run the spark job in k8s

```yml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-minio-app
  namespace: spark-operator
spec:
  sparkConf:
    spark.kubernetes.allocation.batch.size: "50"
  hadoopConf:
    "fs.s3a.endpoint": "http://127.0.0.1:9000"
    "fs.s3a.access.key": "minio"
    "fs.s3a.secret.key": "minio123"
    "fs.s3a.path.style.access": "true"
    "fs.s3a.impl": "org.apache.hadoop.fs.s3a.S3AFileSystem"
  type: Scala
  sparkVersion: 2.4.5
  mode: cluster
  image: minio/spark:v2.4.5-hadoop-3.1
  imagePullPolicy: Always
  restartPolicy:
      type: OnFailure
      onFailureRetries: 3
      onFailureRetryInterval: 10
      onSubmissionFailureRetries: 5
      onSubmissionFailureRetryInterval: 20

  mainClass: org.apache.spark.examples.JavaWordCount
  mainApplicationFile: "local:///opt/spark/examples/target/original-spark-examples_2.11-2.4.6-SNAPSHOT.jar"
  arguments:
  - "s3a://mytestbucket/mydata"
  driver:
    cores: 1
    coreLimit: "1000m"
    memory: "512m"
    labels:
      version: 2.4.5
    sidecars:
    - name: minio-lb
      image: "minio/sidekick:v1.0.0"
      imagePullPolicy: Always
      args: ["--health-path", "/minio/health/ready", "--address", ":9000", "http://minio-distributed-{0...3}.minio-distributed-svc.spark-operator.svc.cluster.local:9000"]
      ports:
        - containerPort: 9000

  executor:
    cores: 1
    instances: 4
    memory: "512m"
    labels:
      version: 2.4.5
    sidecars:
    - name: minio-lb
      image: "minio/sidekick:v1.0.0"
      imagePullPolicy: Always
      args: ["--health-path", "/minio/health/ready", "--address", ":9000", "http://minio-distributed-{0...3}.minio-distributed-svc.spark-operator.svc.cluster.local:9000"]
      ports:
        - containerPort: 9000
```

```
kubectl create -f spark-job.yaml
kubectl logs -f --namespace spark-operator spark-minio-app-driver spark-kubernetes-driver
```

#### Monitor

- Health-Check: Health check is provided at the path "/v1/health". It returns "200 OK" even if any one of the sites is reachable, else it returns "502 Bad Gateway" error.

[gh-downloads]: https://img.shields.io/github/downloads/minio/sidekick/total?color=pink&label=GitHub%20Downloads
