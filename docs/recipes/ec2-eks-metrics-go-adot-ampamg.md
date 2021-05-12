# Using AWS Distro for OpenTelemetry in EKS to ingest metrics into AMP

In this recipe we show you how to instrument a Go application and
use AWS Distro for OpenTelemetry (ADOT) to ingest metrics into
Amazon Managed Service for Prometheus (AMP).
Then we're using AMG to visualize the metrics.

## Infrastructure

### Architecture

The ADOT-AMP pipeline enables us to use the ADOT Collector to scrape a Prometheus-instrumented application, and send the scraped metrics to AWS Managed Service for Prometheus (AMP). 

![Architecture](https://aws-otel.github.io/static/Prometheus_Pipeline-07344e5466b05299cff41d09a603e479.png)

The ADOT-AMP pipeline includes two OpenTelemetry Collector components specific to Prometheus — the Prometheus Receiver and the AWS Prometheus Remote Write Exporter. 

[source](https://aws-otel.github.io/docs/getting-started/prometheus-remote-write-exporter)

### Setup an EKS cluster

For this recipe we require a EKS cluster to be available.
You can either use an existing EKS cluster or create one using the following template [file](ec2-eks-metrics-go-adot-ampamg/cluster_config.yaml).

This template will create a new cluster with [AWS fargate](https://aws.amazon.com/fargate/) enabled. 

Edit the template file and set your region to one of the available regions for Amazon Managed Prometheus service:

* us-east-1
* us-east-2
* us-west-2
* eu-central-1
* eu-west-1

Make sure to overwrite this region in your bash session for example:
```
export AWS_DEFAULT_REGION=eu-west-1
```
Other regions are currently unsupported for Amazon Managed Prometheus service.

Make sure you have installed the [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) command in your environment

Create your cluster using the following command.
```
eksctl create cluster -f cluster-config.yaml
```

### Setup an ECR repository

Create an ECR repository for the sample application

```
aws ecr create-repository \
    --repository-name prometheus-sample-app \
    --image-scanning-configuration scanOnPush=true \
    --region eu-west-1
```

### Setup Amazon Managed Service for Prometheus (AMP) 

#### Create Amazon Managed Service for Prometheus workspace

[External: Amazon Managed Service for Prometheus: Getting started](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-getting-started.html)

create a workspace using the AWS CLI 
```
aws amp create-workspace --alias prometheus-sample-app
```
Take note of the returned workspace-id

Verify the workspace is created using:
```
aws amp list-workspaces
```

#### Setup IAM role permisions for scraping

1. Create the IAM role for the service account by following the steps in [Set up service roles for the ingestion of metrics from Amazon EKS clusters](https://docs.aws.amazon.com/prometheus/latest/userguide/set-up-irsa1.html#set-up-irsa-ingest). The ADOT Collector will use this role when it scrapes and exports metrics

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
[source](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-OpenTelemetry.html)


### Setup AWS Distro for OpenTelemetry (ADOT) Collector 

[Reference: Guide](https://aws-otel.github.io/docs/getting-started/prometheus-remote-write-exporter/eks#aws-distro-for-opentelemetry-adot-collector-setup)

#### Download your template file

Depending on if your EKS cluster uses fargate or not we will use a different template file for the rest of this guide.

The following template file uses a deamonset for it's configuration (docs/recipes/ec2-eks-metrics-go-adot-ampamg)[prometheus-daemonset.yaml].

Fargate does not support deamonsets so you can use the following template if your cluster was setup with fargate.


#### Edit your template file

In this example, the ADOT Collector configuration uses an annotation (scrape=true) to tell which target endpoints to scrape. This allows the ADOT Collector to distinguish the sample app endpoint from kube-system endpoints in your cluster. You can remove this from the re-label configurations if you want to scrape a different sample app. 

Use the following steps to edit the downloaded file for your environment:

1\. Replace **<REGION\>** with your current Region. 

2\. Replace **<YOUR_ENDPOINT>**  with your prometheus workspace endpoint URL.

Get your prometheus endpoint url by executing the following query:
```
$aws amp describe-workspace --workspace-id `aws amp list-workspaces --alias prometheus-sample-app --query 'workspaces[*].workspaceId' --output text` --query 'workspace.prometheusEndpoint'
```

3\. Finally replace your <YOUR_ACCOUNT_ID>  with your current account ID.

The following command will return the account ID for the current session:
```
aws sts get-caller-identity --query Account --output text
```
<br/>

#### Deploy the template to your cluster

Make sure you have [installed](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) and configured the kubectl command.

```
$ kubectl apply -f prometheus-fargate.yaml
```

In case you used the deamonset template use the following command
```
$ kubectl apply -f prometheus-daemonset.yaml
```


You can verify that the ADOT Collector has started with this command:

```
$ kubectl get pods -n adot-col
```

### Setup Amazon managed graphana

[Amazon Managed Service for Grafana – Getting Started](https://aws.amazon.com/blogs/mt/amazon-managed-grafana-getting-started/)

Setup a new AMG manged workspace using the Getting Started guide above.

Make sure to add "Amazon Managed Service for Prometheus" as a datasource.


## Application

The following sample application will be used in this guide.

[Sample App Github](https://github.com/aws-observability/aws-otel-community/tree/master/sample-apps/prometheus)


Clone the following Git repository
```
$ git clone git@github.com:aws-observability/aws-otel-community.git
```

Build the container
```
$ cd ./aws-otel-community/sample-apps/prometheus
$ docker build . -t prometheus-sample-app:latest
```

Authenticate to your default registry:

```
aws ecr get-login-password --region region | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.region.amazonaws.com
```

Push container to the ECR repository
```
docker tag prometheus-sample-app:latest <aws_account_id>.dkr.ecr.eu-west-1.amazonaws.com/prometheus-sample-app:latest
docker push <aws_account_id>.dkr.ecr.eu-west-1.amazonaws.com/prometheus-sample-app:latest
```

Edit ec2-eks-metrics-go-adot-ampamg.md to contain your ECR image path.

Deploy the sample app to your cluster:
```
$ kubectl apply -f prometheus-sample-app.yaml
```

## End-to-end


### Verify your pipeline is working 

Enter the following command to and not down the name of the collector pod:

```
kubectl get pods -n adot-col
```

Verify that the pipeline works by using the logging exporter. Our example template is already integrated with the logging exporter. Enter the following commands.

```
kubectl get pods -A
kubectl logs -n adot-col name_of_your_adot_collector_pod
```

Some of the scraped metrics from the sample app will look like the following example. 

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
$AMP_ENPOINT = awscurl --service="aps" --region="AMP_REGION" "https://AMP_ENDPOINT/api/v1/query?query=adot_test_gauge0"
{"status":"success","data":{"resultType":"vector","result":[{"metric":{"__name__":"adot_test_gauge0"},"value":[1606512592.493,"16.87214000011479"]}]}}
```

### Create a grafana dashboard

[User Guide: Dashboards](https://docs.aws.amazon.com/grafana/latest/userguide/dashboard-overview.html)

[Best practices for creating dashboards](https://grafana.com/docs/grafana/latest/best-practices/best-practices-for-creating-dashboards/)

@todo: Example dashboard?

## Cleanup
@todo
