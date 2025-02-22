# Custom Python3 Version on EMR

EMR 6.x uses Amazon Linux 2, on which Python 3.7.16 is the default Python3 version.

Additional versions of Python3 can be installed in different ways depending on your requirements:

- [1. Installed as a seperate python version in `/usr/local`](#1-install-separate-python-version-in-usrlocal)
- [2. Installed as a container image on YARN](#2-container-images-on-yarn)

This example documents the above options, as well as benefits and limitations.

> **Note**: We also take advantage of this opportunity to upgrade OpenSSL as [urllib3 v2.0 only supports OpenSSL 1.1.1+](https://urllib3.readthedocs.io/en/latest/v2-migration-guide.html#common-upgrading-issues)

## Requirements

All of the below commands assume that you have:
- An IAM user that can create EMR Clusters
- Appropriate [runtime and service roles for EMR](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-plan-access-iam.html)
- Uploaded the necessary scripts to an S3 bucket of your choosing

For step #3, you can clone this repository and use `aws s3 sync` to upload the sample scripts.

```bash
S3_BUCKET=<your-bucket-name>
git clone https://github.com/aws-samples/aws-emr-utilities.git
cd utilities/emr-ec2-custom-python3
aws s3 sync custom-python s3://${S3_BUCKET}/code/bootstrap/custompython/
```

For our experiments, we're going to use the latest `bugfix` version of Python and a sample PySpark script.

### 1. Install separate python version in `/usr/local`

This is the simplest and most straight-forward approach, but also requires you to re-install any Python modules preinstalled on EMR you might use.

By using this approach, you likely won't impact other services or code on the system that relies on Python3.7.

> **Note**: Bootstrap actions can add additional provisioning time to your EMR Cluster, particularly if you do something like compiling Python. See [Using a custom AMI](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-custom-ami.html) for how to mitigate this.

We'll start a cluster with two primary changes:
1. Provide a link to our bootstrap action uploaded above to install Python. See the setting `--bootstrap-actions` as below.
2. Customize our Spark environment to point to the new Python installation. See the setting `--configurations`.

```bash
aws emr create-cluster \
 --name "spark-custom-python" \
 --region ${AWS_REGION} \
 --bootstrap-actions Path="s3://${S3_BUCKET}/code/bootstrap/custompython/install-python.sh"  \
 --log-uri "s3n://${S3_BUCKET}/logs/emr/" \
 --release-label "emr-6.10.0" \
 --use-default-roles \
 --applications Name=Spark Name=Livy Name=JupyterEnterpriseGateway \
 --instance-fleets '[{"Name":"Primary","InstanceFleetType":"MASTER","TargetOnDemandCapacity":1,"TargetSpotCapacity":0,"InstanceTypeConfigs":[{"InstanceType":"c5a.2xlarge"},{"InstanceType":"m5a.2xlarge"},{"InstanceType":"r5a.2xlarge"}]},{"Name":"Core","InstanceFleetType":"CORE","TargetOnDemandCapacity":0,"TargetSpotCapacity":1,"InstanceTypeConfigs":[{"InstanceType":"c5a.2xlarge"},{"InstanceType":"m5a.2xlarge"},{"InstanceType":"r5a.2xlarge"}],"LaunchSpecifications":{"OnDemandSpecification":{"AllocationStrategy":"lowest-price"},"SpotSpecification":{"TimeoutDurationMinutes":10,"TimeoutAction":"SWITCH_TO_ON_DEMAND","AllocationStrategy":"capacity-optimized"}}}]' \
 --scale-down-behavior "TERMINATE_AT_TASK_COMPLETION" \
 --auto-termination-policy '{"IdleTimeout":14400}' \
 --configurations '[{"Classification":"spark-env","Configurations":[{"Classification":"export","Properties":{"PYSPARK_PYTHON": "/usr/local/python3.11.3/bin/python3.11"}}]}]'
```

Now, any Spark job you submit will automatically use the `PYSPARK_PYTHON` you provided. Let's run a _very_ simple test script that just prints out our Python version. 

```python
import sys

from pyspark.sql import SparkSession

spark = (
    SparkSession.builder.appName("Python Spark SQL basic example")
    .getOrCreate()
)

assert (sys.version_info.major, sys.version_info.minor) == (3, 11)
```

```bash
# Use the Cluster ID from the previous create-cluster command or console. For example:j-2W2SS0V0RKG96
CLUSTER_ID=${YOUR_ID} 

aws emr add-steps \
    --cluster-id ${CLUSTER_ID} \
    --steps Type=CUSTOM_JAR,Name="Spark Program",Jar="command-runner.jar",ActionOnFailure=CONTINUE,Args="[spark-submit,--deploy-mode,client,s3://${S3_BUCKET}/code/bootstrap/custompython/validate-python-version.py]"
```

The step should complete successfully!

#### Reducing Cluster Start Time

As mentioned above, while compiling Python as a bootstrap action may work for long-lived clusters, it's not ideal for ephemeral clusters. There are a few options here.

1. [Use a custom AMI](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-custom-ami.html)
2. Build once and copy the resulting artifact using a bootstrap action
3. Roll your own RPM

As #1 is already well documented and #3 requires some deep Linux expertise, we'll show how to do #2. 

Let's assume you used the bootstrap action above and have your Python installation in `/usr/local/python3.11.3`. Connect to your EMR cluster (I prefer SSM, you may have SSH enabled) and perform the following actions.

- Create an archive of the installation

```bash
cd /usr/local
tar czvf python3.11.tar.gz python3.11.3/
```

- Copy it up to S3

```bash
aws s3 cp python3.11.tar.gz s3://${S3_BUCKET}/artifacts/emr/
```

You can now download and unpack this archive in new cluster installations. See the [`copy-python.sh`](./custom-python/copy-python.sh) script for details - if you performed the `aws sync` command above, you'll already have this in your S3 bucket. Use the same `create-cluster` command, but replace `--bootstrap-actions` with this script and pass in an argument of the archive location.

```
--bootstrap-actions Path="s3://${S3_BUCKET}/code/bootstrap/custompython/copy-python.sh",Args=\[s3://${S3_BUCKET}/artifacts/emr/python3.11.tar.gz\]
```

The script will download and unpack the Python installation in `/usr/local`. It should only add ~10 seconds to cluster start time.

### 2. Container Images on YARN

You can also make use of container images to completely isolate your Python environment. 

This repository contains a complete [CodeBuild Pipeline template](container-image/codebuild-docker.cf) you can use to build and publish a Dockerfile with Python 3.11 and PyArrow. 

For additional details, see the [EMR Spark with Docker](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-spark-docker.html) and [CodeBuild Docker sample](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html) docs.

We'll do this step-by-step, too. 

#### Spark, Docker, and EMR step-by-step

First we set a few variables we need. We use the AWS CLI and Docker.

```bash
AWS_REGION=us-west-2 #change to your region
ACCOUNT_ID=$(aws sts get-caller-identity --output text --query "Account")
LOG_BUCKET=aws-logs-${ACCOUNT_ID}-${AWS_REGION}
#login to ECR
ECR_URL=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_URL
DOCKER_IMAGE_NAME=${ECR_URL}/emr-docker-examples:pyspark-example
```

Then we create a basic EMR cluster with some important configurations.

- `ebs-root-volume-size` increased to 100 - Container images are stored on the root volume by default.
- `container-executor` configuration classification that adds your ECR repository as a trusted registry.
- `spark-defaults` to specify `dockeras as the yarn container runtime and default all Spark jobs to our image.

```bash
aws emr create-cluster \
    --name "emr-docker-python3-spark" \
    --region ${AWS_REGION} \
    --release-label emr-6.10.0 \
    --log-uri "s3n://${LOG_BUCKET}/elasticmapreduce/" \
    --ebs-root-volume-size 100 \
    --applications Name=Spark Name=Livy Name=JupyterEnterpriseGateway \
    --use-default-roles \
    --instance-fleets '[{"Name":"Primary","InstanceFleetType":"MASTER","TargetOnDemandCapacity":1,"TargetSpotCapacity":0,"InstanceTypeConfigs":[{"InstanceType":"c5a.2xlarge"},{"InstanceType":"m5a.2xlarge"},{"InstanceType":"r5a.2xlarge"}]},{"Name":"Core","InstanceFleetType":"CORE","TargetOnDemandCapacity":0,"TargetSpotCapacity":1,"InstanceTypeConfigs":[{"InstanceType":"c5a.2xlarge"},{"InstanceType":"m5a.2xlarge"},{"InstanceType":"r5a.2xlarge"}],"LaunchSpecifications":{"OnDemandSpecification":{"AllocationStrategy":"lowest-price"},"SpotSpecification":{"TimeoutDurationMinutes":10,"TimeoutAction":"SWITCH_TO_ON_DEMAND","AllocationStrategy":"capacity-optimized"}}}]' \
    --auto-termination-policy '{"IdleTimeout":14400}' \
    --configurations '[
        {
            "Classification": "container-executor",
            "Configurations": [
                {
                    "Classification": "docker",
                    "Properties": {
                        "docker.trusted.registries": "'${ACCOUNT_ID}'.dkr.ecr.'${AWS_REGION}'.amazonaws.com",
                        "docker.privileged-containers.registries": "'${ACCOUNT_ID}'.dkr.ecr.'${AWS_REGION}'.amazonaws.com"
                    }
                }
            ]
        },
        {
            "Classification":"spark-defaults",
            "Properties":{
                "spark.executorEnv.YARN_CONTAINER_RUNTIME_TYPE":"docker",
                "spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_TYPE":"docker",
                "spark.executorEnv.YARN_CONTAINER_RUNTIME_DOCKER_IMAGE":"'${DOCKER_IMAGE_NAME}'",
                "spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_DOCKER_IMAGE":"'${DOCKER_IMAGE_NAME}'",
                "spark.executor.instances":"2"
            }
        }
    ]'
```

While that's starting, let's create a simple container image. To keep things small, we'll use `python:3.11-slim` as the base image and copy over OpenJDK 17 from `eclipse-temurin:17`.

**container-image/Dockerfile:**
```dockerfile
FROM python:3.11-slim AS base

# Copy OpenJDK 17
ENV JAVA_HOME=/opt/java/openjdk
COPY --from=eclipse-temurin:17 $JAVA_HOME $JAVA_HOME
ENV PATH="${JAVA_HOME}/bin:${PATH}"

# Upgrade pip
RUN pip3 install --upgrade pip

# Configure PySpark
ENV PYSPARK_DRIVER_PYTHON python3
ENV PYSPARK_PYTHON python3

# Install pyarrow
RUN pip3 install pyarrow==12.0.0
```

Now build the image.

```bash
cd utilities/emr-ec2-custom-python3
docker build -t local/pyspark-example -f container-image/Dockerfile .
```

And you should be able to run a quick test.

```bash
docker run --rm -it local/pyspark-example python -c "import pyarrow; print(pyarrow.__version__)"
```

This should output `12.0.0`.

Finally, we can create a repository in ECR and tag and push our image there.

```bash
# login to ECR
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URL}
# create repo
aws ecr create-repository --repository-name emr-docker-examples --image-scanning-configuration scanOnPush=true
# push to ECR
docker tag local/pyspark-example ${DOCKER_IMAGE_NAME}
docker push ${DOCKER_IMAGE_NAME}
```

You can now upload a sample PySpark script and run it on EMR. Because you sepcified the image name in the cluster configuration, you don't need to specify it in your `spark-submit` (although you can if you want to use a different image).

```
aws s3 cp container-image/docker-numpy.py s3://${S3_BUCKET}/code/pyspark/container-example/
```

```bash
# Use the Cluster ID from the previous create-cluster command or console. For example:j-2845LE9NEI32S
CLUSTER_ID=${YOUR_ID}

aws emr add-steps \
    --cluster-id ${CLUSTER_ID} \
    --steps Type=CUSTOM_JAR,Name="Spark Program",Jar="command-runner.jar",ActionOnFailure=CONTINUE,Args="[spark-submit,--deploy-mode,client,s3://${S3_BUCKET}/code/pyspark/container-example/docker-numpy.py]"
```

And that's it! You should be able to navigate to the "Steps" portion of the EMR Console and see the `stdout` from your job above.

> **Note**: I used `--deploy-mode client` above to ensure that the EMR Steps API can record output from the example code. In most cases you want to use `cluster` as that will distribute the Spark driver to Worker nodes. 