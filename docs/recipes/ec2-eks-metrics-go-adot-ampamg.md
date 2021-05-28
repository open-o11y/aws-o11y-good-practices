# Using AWS Distro for OpenTelemetry in EKS to ingest metrics into AMP

In this recipe we show you how to instrument a Go application and
use [AWS Distro for OpenTelemetry (ADOT)](https://aws.amazon.com/otel) to ingest metrics into
[Amazon Managed Service for Prometheus (AMP)](https://aws.amazon.com/prometheus/) .
Then we're using [Amazon Managed Service for Grafana (AMG)](https://aws.amazon.com/grafana/) to visualize the metrics.

We will be setting up an [Amazon Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/) cluster and [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/) repository to demonstrate a complete scenario.

## Infrastructure

### Architecture

The ADOT-AMP pipeline enables us to use the ADOT Collector to scrape a Prometheus-instrumented application, and send the scraped metrics to AMP. 

![Architecture](https://aws-otel.github.io/static/Prometheus_Pipeline-07344e5466b05299cff41d09a603e479.png)

The ADOT-AMP pipeline includes two OpenTelemetry Collector components specific to Prometheus — the Prometheus Receiver and the AWS Prometheus Remote Write Exporter. 

[Getting Started with Prometheus Remote Write Exporter for AMP](https://aws-otel.github.io/docs/getting-started/prometheus-remote-write-exporter)


### Prerequisites

* The AWS CLI is [installed](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) in your environment.
* You need to install the [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) command in your environment.
* You need to install [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) in your environment. 

### Setup an EKS cluster

For this recipe we require a EKS cluster to be available.
You can either use an existing EKS cluster or create one using [cluster_config.yaml](ec2-eks-metrics-go-adot-ampamg/cluster_config.yaml).

This template will create a new cluster with [AWS Fargate](https://aws.amazon.com/fargate/) enabled. 

Edit the template file and set your region to one of the available regions for AMP:

* `us-east-1`
* `us-east-2`
* `us-west-2`
* `eu-central-1`
* `eu-west-1`

Make sure to overwrite this region in your bash session for example:
```
export AWS_DEFAULT_REGION=eu-west-1
```
Other regions are currently unsupported for AMP.

Create your cluster using the following command.
```
eksctl create cluster -f cluster-config.yaml
```

### Setup an ECR repository

Create an ECR repository for the sample application.

```
aws ecr create-repository \
    --repository-name prometheus-sample-app \
    --image-scanning-configuration scanOnPush=true \
    --region eu-west-1
```

### Setup AMP 

#### Create AMP workspace

create a workspace using the AWS CLI 
```
aws amp create-workspace --alias prometheus-sample-app
```
Take note of the returned workspace-id

Verify the workspace is created using:
```
aws amp list-workspaces
```
For more details check out the [AMP Getting started](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-getting-started.html) guide.

#### Setup IAM role permissions for scraping

1. Create the IAM role for the service account by following the steps in [Set up service roles for the ingestion of metrics from Amazon EKS clusters](https://docs.aws.amazon.com/prometheus/latest/userguide/set-up-irsa.html#set-up-irsa-ingest). The AWS Distro for Open Telemetry Collector will use this role when it scrapes and exports metrics

2. Next, edit the trust policy. Open the IAM console at at https://console.aws.amazon.com/iam/

3. In the left navigation pane choose Roles and find the amp-iamproxy-ingest-role that you created in step 1.

4. Choose the Trust relationships tab and choose Edit trust relationship.

5. In the trust relationship policy JSON, replace aws-amp with adot-col and then choose Update Trust Policy. Your resulting trust policy should look like the following: 
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::account-id:oidc-provider/oidc.eks.aws_region.amazonaws.com/id/openid"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.aws_region.amazonaws.com/id/openid:sub": "system:serviceaccount:adot-col:amp-iamproxy-ingest-service-account"
        }
      }
    }
  ]
}
```
6. Choose the Permissions tab and make sure that the following permissions policy is attached to the role. 
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "aps:RemoteWrite",
                "aps:GetSeries",
                "aps:GetLabels",
                "aps:GetMetricMetadata"
            ],
            "Resource": "*"
        }
    ]
}
```
[Reference: Set up metrics ingestion using AWS Distro for Open Telemetry](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-OpenTelemetry.html)

### Setup ADOT Collector 

#### Download your template file

Depending on if your EKS cluster uses AWS Fargate or not we will use a different template file for the rest of this guide.

The following template file uses a deamonset for it's configuration [prometheus-daemonset.yaml](ec2-eks-metrics-go-adot-ampamg/prometheus-daemonset.yaml).

AWS Fargate does not support deamonsets so you can use the following template [prometheus-fargate.yaml](ec2-eks-metrics-go-adot-ampamg/prometheus-fargate.yaml), if your cluster was setup with AWS Fargate.


#### Edit your template file

In this example, the ADOT Collector configuration uses an annotation (scrape=true) to tell which target endpoints to scrape. This allows the ADOT Collector to distinguish the sample app endpoint from kube-system endpoints in your cluster. You can remove this from the re-label configurations if you want to scrape a different sample app. 

Use the following steps to edit the downloaded file for your environment:

1\. Replace **<REGION\>** with your current Region. 

2\. Replace **<YOUR_ENDPOINT\>**  with your AMP workspace endpoint URL.

Get your AMP endpoint url by executing the following query:
```
aws amp describe-workspace \ 
    --workspace-id `aws amp list-workspaces --alias prometheus-sample-app --query 'workspaces[0].workspaceId' --output text` \
    --query 'workspace.prometheusEndpoint'
```

3\. Finally replace your **<YOUR_ACCOUNT_ID\>**  with your current account ID.

The following command will return the account ID for the current session:
```
aws sts get-caller-identity --query Account --output text
```
<br/>

#### Deploy the template to your cluster

```
kubectl apply -f prometheus-fargate.yaml
```

In case you used the deamonset template use the following command
```
kubectl apply -f prometheus-daemonset.yaml
```

You can verify that the ADOT Collector has started with this command:

```
kubectl get pods -n adot-col
```

For more information check out the [AWS Distro for OpenTelemetry (ADOT) Collector Setup](https://aws-otel.github.io/docs/getting-started/prometheus-remote-write-exporter/eks#aws-distro-for-opentelemetry-adot-collector-setup)

### Setup AMG

[Amazon Managed Service for Grafana – Getting Started](https://aws.amazon.com/blogs/mt/amazon-managed-grafana-getting-started/)

Setup a new AMG workspace using the Getting Started guide above.

Make sure to add "Amazon Managed Service for Prometheus" as a datasource during creation.

![Service managed permission settings](https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2020/12/09/image008-1024x870.jpg)


## Application

The following sample application will be used in this guide:

[Sample App Github](https://github.com/aws-observability/aws-otel-community/tree/master/sample-apps/prometheus)


### Build
Clone the following Git repository
```
git clone git@github.com:aws-observability/aws-otel-community.git
```

Build the container
```
cd ./aws-otel-community/sample-apps/prometheus
docker build . -t prometheus-sample-app:latest
```

If go mod fails in your environment due to a proxy.golang.or i/o timeout,
you are able to bypass the go mod proxy by editing the Dockerfile.

Change the following line in the Docker file:
```
RUN GO111MODULE=on go mod download
```
to:
```
RUN GOPROXY=direct GO111MODULE=on go mod download
```

### Push
Change the region variable to the region you selected in the beginning of this guide:
```
export REGION="eu-west-1"
export ACCOUNTID=`aws sts get-caller-identity --query Account --output text`
```

Authenticate to your default registry:
```
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin "$ACCOUNTID.dkr.ecr.$REGION.amazonaws.com"
```

Push container to the ECR repository
```
docker tag prometheus-sample-app:latest "$ACCOUNTID.dkr.ecr.$REGION.amazonaws.com/prometheus-sample-app:latest"
docker push "$ACCOUNTID.dkr.ecr.$REGION.amazonaws.com/prometheus-sample-app:latest"
```

### Deploy
Edit prometheus-sample-app.yaml to contain your ECR image path.

Deploy the sample app to your cluster:
```
kubectl apply -f prometheus-sample-app.yaml
```

## End-to-end

Now that you have the infrastructure and the application in place, we will test out the setup, sending metrics from the Go app running in EKS to AMP and visualize it in AMG.

### Verify your pipeline is working 

Enter the following command to and not down the name of the collector pod:

```
kubectl get pods -n adot-col
```

You should be able to see a adot-collector pod in the running state:
```
NAME                              READY   STATUS    RESTARTS   AGE
adot-collector-5f7448f6f6-cj7j8   1/1     Running   0          1h
```

Note down the name of this pod. Our example template is already integrated with the logging exporter. Enter the following command:

```
kubectl logs -n adot-col <name_of_your_adot_collector_pod>
```

Some of the scraped metrics from the sample app will look like the following example: 
```
Resource labels:
     -> service.name: STRING(kubernetes-service-endpoints)
     -> host.name: STRING(192.168.16.238)
     -> port: STRING(8080)
     -> scheme: STRING(http)
InstrumentationLibraryMetrics #0
Metric #0
Descriptor:
     -> Name: test_gauge0
     -> Description: This is my gauge
     -> Unit: 
     -> DataType: DoubleGauge
DoubleDataPoints #0
StartTime: 0
Timestamp: 1606511460471000000
Value: 0.000000
```

To test whether AMP received the metrics, use awscurl. This tool enables you to send HTTP requests through the command line with AWS Sigv4 authentication, so you must have AWS credentials set up locally with the correct permissions to query from AMP For instructions on installing awscurl, see awscurl.

In the following command, and AMP_ENDPOINT with the information for your AMP workspace. 

```
awscurl --service="aps" \ 
    --region="AMP_REGION" "https://AMP_ENDPOINT/api/v1/query?query=adot_test_gauge0" \
        {"status":"success","data":{"resultType":"vector","result":[{"metric":{"__name__":"adot_test_gauge0"},"value":[1606512592.493,"16.87214000011479"]}]}}
```

### Create a Grafana dashboard in AMG

Use the following guides to create your first dashboard:

* [User Guide: Dashboards](https://docs.aws.amazon.com/grafana/latest/userguide/dashboard-overview.html)
* [Best practices for creating dashboards](https://grafana.com/docs/grafana/latest/best-practices/best-practices-for-creating-dashboards/)

![placeholder-image](https://d1.awsstatic.com/products/grafana/amg-console-1.a9bcc3ab4dc86a378eb808851f54cee8a34cb300.png)

## Cleanup

1. Remove the cluster
```
eksctl delete cluster --name amp-eks-fargate
```
2. Remove the AMP workspace
```
aws amp delete-workspace --workspace-id `aws amp list-workspaces --alias prometheus-sample-app --query 'workspaces[0].workspaceId' --output text`
```
3. Remove the amp-iamproxy-ingest-role IAM role 
```
aws delete-role --role-name amp-iamproxy-ingest-role
```
4. Remove the AMG workspace by removing it from the console. 