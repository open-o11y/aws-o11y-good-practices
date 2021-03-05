# Telemetry

Telemetry is all about how the signals are collected from various sources,
including your own app and infrastructure and routed to destinations where
they are consumed:

![telemetry concept](images/telemetry.png)

Let's further dive into the concepts introduced in above figure.

!!! question "Routing? Shipping? Ingesting?"
    There are many terms out there people use to refer to the process of
    getting the signals from sources to destinations including routing,
    shipping, aggregation, ingesting etc. and while they may mean slightly 
    different things, we will use them here interchangeably.

## Sources

Where signals come from.

Two types: under your control (application source code, instrumentation), and
the ones provided by AWS via an API.

## Routers and Agents

Where signals shoud go, optional filtering and aggregation.



## Destinations

Where signals end up, for consumption. 
