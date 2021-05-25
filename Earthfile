ARG spark_version=3.1.1
ARG hadoop_version=3.2.0
ARG hive_version=2.3.7
ARG maven_version=3.6.3

# Spark runtime build options
ARG scala_version=2.12
ARG aws_java_sdk_version=1.11.797

ARG container_repo=viaductai/spark
ARG container_tag_prefix="${latest:+${latest}}spark-${spark_version}_hadoop-${hadoop_version}_hive-${hive_version}_k8s_aws"
ARG container_tag_suffix="-${EARTHLY_TARGET_TAG_DOCKER}"
ARG container_tag="${container_tag_prefix}${container_tag_suffix}"

build-deps:
  # use python base image because spark build process requires python
  FROM python:3.7-slim-stretch

  # TODO: use openjdk-11
  RUN apt-get update \
    && echo "deb http://ftp.us.debian.org/debian sid main" >> /etc/apt/sources.list \
    && mkdir -p /usr/share/man/man1 \
    && apt-get install -y git curl wget openjdk-8-jdk patch \
    && rm -rf /var/cache/apt/*

  # maven
  RUN cd /opt \
    &&  wget https://downloads.apache.org/maven/maven-3/${maven_version}/binaries/apache-maven-${maven_version}-bin.tar.gz \
    &&  tar zxvf /opt/apache-maven-${maven_version}-bin.tar.gz \
    &&  rm apache-maven-${maven_version}-bin.tar.gz

  ENV PATH=/opt/apache-maven-${maven_version}/bin:$PATH
  ENV MAVEN_HOME /opt/apache-maven-${maven_version}

  # configure the pentaho nexus repo to prevent build errors
  # similar to the following: https://github.com/apache/hudi/issues/2479
  COPY ./maven-settings.xml ${MAVEN_HOME}/conf/settings.xml

build-glue-hive-client:
  FROM +build-deps

  # Download and extract Apache hive source
  RUN wget https://github.com/apache/hive/archive/rel/release-${hive_version}.tar.gz -O hive.tar.gz
  RUN mkdir hive && tar xzf hive.tar.gz --strip-components=1 -C hive

  ## Build patched hive 2.3.7
  # https://github.com/awslabs/aws-glue-data-catalog-client-for-apache-hive-metastore/issues/26
  WORKDIR /hive
# Patch copied from: https://issues.apache.org/jira/secure/attachment/12958418/HIVE-12679.branch-2.3.patch
  COPY ./aws-glue-spark-hive-client/HIVE-12679.branch-2.3.patch hive.patch
  RUN patch -p0 <hive.patch &&\
    mvn clean install -DskipTests

  # Now with hive patched and installed, build the glue client
  RUN git clone https://github.com/viaduct-ai/aws-glue-data-catalog-client-for-apache-hive-metastore /catalog

  WORKDIR /catalog

  RUN mvn clean package \
    -DskipTests \
    -Dhive2.version=${hive_version} \
    -Dhadoop.version=${hadoop_version} \
    -Daws.sdk.version=${aws_java_sdk_version} \
    -pl -aws-glue-datacatalog-hive2-client

build-spark:
  FROM +build-glue-hive-client

  ENV MAKEFLAGS -j 4

  # Build spark
  WORKDIR /

  RUN git clone https://github.com/apache/spark.git --branch v${spark_version} --single-branch && \
      cd /spark && \
      ./dev/make-distribution.sh \
        --name spark \
        --pip \
        -DskipTests \
        -Pkubernetes \
        -Phadoop-cloud \
        -P"hadoop-${hadoop_version%.*}" \
        -Dhadoop.version="${hadoop_version}" \
        -Dhive.version="${hive_version}" \
        -Phive \
        -Phive-thriftserver

  # copy the glue client jars to spark jars directory
  RUN find /catalog -name "*.jar" | grep -Ev "test|original" | xargs -I{} cp {} /spark/dist/jars/

  RUN rm /spark/dist/jars/aws-java-sdk-bundle-*.jar
  RUN wget --quiet https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/${aws_java_sdk_version}/aws-java-sdk-bundle-${aws_java_sdk_version}.jar -P /spark/dist/jars/
  RUN chmod 0644 /spark/dist/jars/aws-java-sdk-bundle*.jar

  # replace with guava version compatible with latest aws-java-sdk-bundle
  RUN rm -f /spark/dist/jars/guava-14.0.1.jar
  RUN wget --quiet https://repo1.maven.org/maven2/com/google/guava/guava/23.0/guava-23.0.jar -P /spark/dist/jars/
  RUN chmod 0644 /spark/dist/jars/guava-23.0.jar

  # save the final spark distribution
  SAVE ARTIFACT /spark/dist

# Build the spark docker images according to the official Docker images published with the project
# (https://github.com/apache/spark/tree/master/resource-managers/kubernetes/docker/src/main/dockerfiles/spark)
# This ensures that the layout of the container is "correct", which is
# important for spark features like graceful node decommissioning (/opt/decom.sh)
build-spark-image:
  FROM openjdk:8-jre-slim

  ARG spark_uid=185
  # Before building the docker image, first build and make a Spark distribution following
  # the instructions in http://spark.apache.org/docs/latest/building-spark.html.
  # If this docker file is being used in the context of building your images from a Spark
  # distribution, the docker build command should be invoked from the top level directory
  # of the Spark distribution. E.g.:
  # docker build -t spark:latest -f kubernetes/dockerfiles/spark/Dockerfile .

  RUN set -ex && \
      sed -i 's/http:\/\/deb.\(.*\)/https:\/\/deb.\1/g' /etc/apt/sources.list && \
      apt-get update && \
      ln -s /lib /lib64 && \
      apt install -y bash tini libc6 libpam-modules krb5-user libnss3 procps && \
      mkdir -p /opt/spark && \
      mkdir -p /opt/spark/examples && \
      mkdir -p /opt/spark/work-dir && \
      mkdir -p /opt/spark/conf && \
      touch /opt/spark/RELEASE && \
      rm /bin/sh && \
      ln -sv /bin/bash /bin/sh && \
      echo "auth required pam_wheel.so use_uid" >> /etc/pam.d/su && \
      chgrp root /etc/passwd && chmod ug+rw /etc/passwd && \
      rm -rf /var/cache/apt/*

  COPY +build-spark/dist/jars /opt/spark/jars
  COPY +build-spark/dist/bin /opt/spark/bin
  COPY +build-spark/dist/sbin /opt/spark/sbin
  COPY +build-spark/dist/kubernetes/dockerfiles/spark/entrypoint.sh /opt/
  COPY +build-spark/dist/kubernetes/dockerfiles/spark/decom.sh /opt/
  COPY +build-spark/dist/examples /opt/spark/examples
  COPY +build-spark/dist/kubernetes/tests /opt/spark/tests
  COPY +build-spark/dist/data /opt/spark/data

  # configure aws glue data catalog as the Hive Metastore client
  COPY ./conf/hive-site.xml /opt/spark/conf/hive-site.xml

  ENV SPARK_HOME /opt/spark

  WORKDIR /opt/spark/work-dir
  RUN chmod g+w /opt/spark/work-dir
  RUN chmod a+x /opt/decom.sh

  ENTRYPOINT [ "/opt/entrypoint.sh" ]

  # Install python
  # Reset to root to run installation tasks
  USER 0

  RUN mkdir ${SPARK_HOME}/python
  RUN apt-get update && \
      apt install -y python3 python3-pip && \
      pip3 install --upgrade pip setuptools && \
      # Removed the .cache to save space
      rm -r /root/.cache && rm -rf /var/cache/apt/*

  COPY +build-spark/dist/python/pyspark ${SPARK_HOME}/python/pyspark
  COPY +build-spark/dist/python/lib ${SPARK_HOME}/python/lib

  ENV PATH "${PATH}:${SPARK_HOME}/bin"

  WORKDIR /opt/spark/work-dir
  ENTRYPOINT [ "/opt/entrypoint.sh" ]

# Specify the User that the actual main process will run as
  ARG spark_uid=185
  USER ${spark_uid}

  SAVE IMAGE --push ${container_repo}:${container_tag}
