# Dimensions

In the context of this site we consider the o11y space along six dimensions.
Looking at each dimension independently is beneficial from an synthetic
point-of-view, that is, when you're trying to build out a concrete o11y solution
for a given workload, spanning developer-related aspects such as the programming
language used as well as operational topics, for example the runtime environment
like containers or Lambda functions.

!!! note "What is a signal?"
    When we say signal here we mean any kinds of o11y data and metadata points,
    including log entries, metrics, and traces.

Let's have a look at each of the dimensions one by one:

## Analytics

An umbrella term for all kinds of signal destinations including long term
storage and graphical interfaces that let you consume signals.

## Telemetry

How the signals are collected and routed. [Learn more …](telemetry.md)

## Language

The programming language used. This is mainly relevant for your own code.

## Infra

Any sort of infrastructure-related sources of signals, including AWS
infrastructure, for example [VPC flow logs][vpc-flow-logs]
or [S3 bucket access logs][s3-bucket-logs] and it also includes secondary APIs
such as [Kubernetes API logs][k8s-api-logs].

## Compute unit

The way your code is packaged and scheduled. For example, in Lambda that's a
function and in ECS and EKS that's a container (and tasks and pods
respectively).

## Compute engine

This refers to the base runtime environment, which may (EC2 instance, for
example) or may not (Fargate, Lambda) be your responsibility to maintain.



[vpc-flow-logs]: https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html
[s3-bucket-logs]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/enable-server-access-logging.html
[k8s-api-logs]: https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html

