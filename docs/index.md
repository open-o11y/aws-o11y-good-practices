# Welcome!

We are covering recipes for observability (o11y) solutions at AWS on this site.
This includes managed services such as [Amazon Managed Service for Prometheus][amp]
(AMP), libraries/SDKs and agents, for example as provided by [OpenTelemetry][otel]
or [Fluent Bit][fluentbit]. We want to address the needs of both developers and
infrastructure folks.

The way we think about this space is as follows:
The o11y space as shown above has [six dimensions][dimensions/] you can combine to arrive at
a specifc solution.

| dimension | examples |
------------|-----------
| analytics | [AMP][amp] AMG ES CW X-Ray S3 Jaeger |
| telemetry | ADOT Fluent Bit CW agent X-Ray agent |
| languages | Java Python .NET JavaScript Go Rust |
| infra & persistent data | VPC flow logs Prometheus exporters Docker logs RDS
traces |
| compute unit | Batch ECS EKS Beanstalk Lambda|
| compute engine | Fargate EC2 Lightsail |

For example, you might be looking for a solution to:

!!! question "Examplary solution specification"
    I need a logging solution for a Python app I'm running on EKS on Fargate
    with the goal to store the logs in an S3 bucket for further consumption

1. *Analytics*: S3 bucket for further consumption
1. *Telemetry*: FluentBit
1. *Language*: Python
1. *Infra*: N/A
1. *Compute unit*: Kubernetes (EKS)
1. *Compute engine*: EC2

Not every dimension needs to be specified and sometimes it's hard to decide where
to start. Try different paths and compare the pros and cons of certain recipes.

!!! note "Navigation categories"
    To simplify navigation, we're grouping the six layers into the following
    catagories:

    - **By Compute** covering compute engines and units
    - **By Language** covering languages
    - **By Destination** covering telemetry and analytics

[Learn more about dimensions …](dimensions/)


## How to use

You can either use the top navigation menu to browse to a specific index page,
starting with a rough selection. For example, `By Compute` -> `EKS` ->
`Fargate` -> `Logs`.

Alternatively, you can search the site pressing `/` or the `s` key:

![o11y space](images/search.png)

## How to contribute

Raise issues or send in pull requests against the repo.


[amp]: https://aws.amazon.com/prometheus/ "Amazon Managed Service for Prometheus"
[otel]: https://opentelemetry.io/
[fluentbit]: https://fluentbit.io/
[dimensions]: dimensions.md
