# spark-k8s-aws

Build for a spark on kubernetes-ready docker image configured with notable AWS Dependencies, including:
* An up-to-date AWS SDK capable of supporting [ IRSA ](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)
* [ AWS Glue Data Catalog client for Hive Metastore ](https://github.com/viaduct-ai/aws-glue-data-catalog-client-for-apache-hive-metastore)

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
amazing [ Earthly ](https://earthly.dev) project, to democratize an integrate
spark on kubernetes on aws experience until someone develops a Spark
DataSourceV2 API-compliant Glue Data Catalog implementation (which won't
require the absolute hack of patching hive and building spark from source)

