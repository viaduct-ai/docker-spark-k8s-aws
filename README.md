# spark-k8s-aws

Build for an Apache Spark on kubernetes-ready docker image configured with notable AWS Dependencies, including:
* An up-to-date AWS SDK capable of supporting [ IRSA ](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)
* [ AWS Glue Data Catalog client for Hive Metastore ](https://github.com/viaduct-ai/aws-glue-data-catalog-client-for-apache-hive-metastore)

## Use the Docker Image
This build publishes images to docker hub: https://hub.docker.com/r/viaductoss/spark

```
docker pull viaductoss/spark:3.1.1_hadoop-3.2.0_hive-2.3.7_k8s_aws
```

## Build the Docker Image
Builds are managed using https://earthly.dev

```
earthly --use-inline-cache +build-spark-image
```

Use in your own Earthfile build:
```
my-image:
  FROM +github.com/viaduct-ai/docker-spark-k8s-aws+build-spark-image
  # ...
```

## Why?
If you've ever tried building a spark distribution/image with the AWS Glue Data
Catalog Client for Hive, you know it's a PITA.

This project aims to open source a working docker image, built using the
amazing [ Earthly ](https://earthly.dev) tool, to democratize a more integrated
Apache Spark on Kubernetes on AWS experience until someone develops a Spark
DataSourceV2 API-compliant Glue Data Catalog implementation (instead of
this absolute hack of patching hive and building spark from source)

Many thanks to @bbenzikry for open sourcing their solution to build Spark 3 +
Glue compatible docker images. This project builds on their work.
