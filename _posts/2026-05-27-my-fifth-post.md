---
layout: single
title: "Design a data streaming service"
date: 2026-05-27
author: "Cheng Qu"
categories: [blog]
tags: [Cloud]
---

In a multi-tenancy enterprise data system, each tenant has their own DB and data. A data streaming service is foundational infrastructure that enables the building of other microservices and apps that consume the data, without placing additional load on the production DB. It streams the data in near-real-time to downstream consumers via Kafka. The system ensures that change is captured, delivered and available for other services to react to. This service essentially decouples producers from consumers. The application writes to DB normally, and everyone else subscribe to Kafka topics.

Building such a enterprise level data service becomes difficult for many reasons, to just name a few:
1. The service may need to connect to hundreds of thousands of databases, which introduces challenges related to credential management, tenant onboarding, connection management, schema differences, schema evolution, database health monitoring, and connection monitoring.
2. Change data capture (CDC) complexity. For each tenant, the service must manage partition, track replication offsets, handle failures, recover from outage, and ensure that change events are processed correctly.
3. Data consistency and reliability. Failures can occur across multiple components in the system. The service must ensure correct event ordering, delivery guarantees, and consistent recovery behavior.
4. How to design E2E tests to ensure quality releases.

Given this context, I will provide a high-level overview of the architecture and key design decisions of a data streaming system. While the discussion is loosely inspired by my professional experience, it does not disclose any specific details, source code, or other proprietary information.

**System Architecture:**<br>
Before we jump into detailed design, lets think about what some of the major components this system needs to have. Firstly, the platform requires a robust tenant lifecycle management strategy. When a new tenant is onboarded, the platform must discover the tenant's database location, securely obtain and manage the database credentials, and establish connectivity to the tenant's database. The platform then needs to configure change data capture (CDC) mechanisms—such as triggers, transaction log readers, or database-native replication features—and register the tenant with the replication infrastructure. Finally, it must route the captured change events to the appropriate Kafka topics so that downstream services can consume the data in a scalable and reliable manner. Secondly the system need a strategy for managing kafka configuration changes. The configuration may need to be customized based on data flow and system health. Kafka configurations cannot be treated as static because tenant growth, data volume, throughput requirements, and infrastructure health may change over time. Therefore, the tenant lifecycle and kafka configuration management can be seen as the control plane/layer of the system.

In addition to the control plane, we would need a data plane, which 'moves the data'. The always-on pipeline would continuously streams changes from Postgres to kafka. Furthermore, we need an operational layer for keeping the system healthy. This would include monitoring, maintenance and some self-healing components. We will go into each layer in more details later.

Since this system will likely be made of many stateless Lambdas in each layer, they can't directly talk to each other. We would need to introduce a mechanism to enable the three layers to coordinate. We can call this shared state layer that ties all three architectural layers together. It's essentially the system's distributed memory — replacing what would be in-memory state in a monolithic application. DynamoDB is a strong candidate for this job because it provides state storage and low latency access, it also scales very well and natively supports event driven integration with DynamoDB streams.

![Architectural Design]({{ '/assets/images/3_layer_arch+memory.png' | relative_url }})

**Control Plane:**<br>
Next we can go into detail design for each layer. For tenant lifecycle management, if onboarding and managing tenant isn't automated, the platform won't scale well. We can introduce an orchestration layer to automate tenant lifecycle management. AWS Step Functions is a good fit because it provides a durable, stateful workflow engine that can coordinate the multiple steps required to onboard a tenant. The workflow can retrieve tenant metadata and credentials, validate database connectivity, provision CDC infrastructure, register Kafka topics, and update the tenant registry. Step Functions also provides built-in retry policies, error handling, and workflow visibility, which simplifies the operational management of large numbers of tenant onboarding processes. We can leverage the step function to enable, disable and delete tenants as needed, restart failed connectors, and calculate replication lag if needed.

Since tenant onboarding must be fully automated, we need a mechanism to trigger the onboarding workflow whenever a new tenant is created. One approach is to publish a tenant onboarding event to an SNS topic as part of the customer onboarding process. A Lambda function can subscribe to the SNS topic and initiate the corresponding AWS Step Functions workflow. This decouples the onboarding application from the streaming platform and allows the workflow to evolve independently. Alternatively we can also use eventBridge to start step function, but it lacks the control and flexibility a lambda function can provide, such as validation, data enrichment, and other custom business logic as needed.

AWS Step Functions orchestrates the provisioning of all resources required for data streaming. This includes connecting to the tenant database, creating database objects required for change capture, deploying CDC connectors, configuring Kafka topics and permissions, performing an initial data snapshot, and finally enabling continuous replication. Each step can be executed by dedicated services or Lambda functions, with retries and error handling managed by the workflow.

![Tenant onboarding management]({{ '/assets/images/tenant_onboarding_mgmt.png' | relative_url }})

**Data Plane:**<br>
For the data plane, we need to pick the technology that moves the CDC data from Postgres into Kafka. It needs to be long-running, stateful, always-on service, not an event-driven function. We can pick AWS Fargate because it's the "serverless hosting" for that always-on container (Kafka connect image). Fargate scales horizontally i.e. it can run multiple Connect worker tasks behind a service discovery endpoint. Of course we need to provision the Kafka Connect image which is the Data Plane engine — it's the component that actually moves CDC data from Postgres into Kafka. For each tenant, the Kafka connect runtime will have a JDBC connector that polls the tenant's DB table. Again we can track the health of the connectors per tenant and record its status in the system's memory - dynamoDB table.

![Kafka connect]({{ '/assets/images/kafka_connect.png' | relative_url }})

For managing Kafka configuration changes, the platform should continuously monitor data flow and cluster health, and provide mechanisms to safely evolve Kafka configurations without disrupting producers or consumers. For example, we may need to increase replication factor to improve fault tolerance; we may need to adjust partition for throughput; we may need to adjust retention policy for cost and compliance considerations etc. Changes should be applied through controlled, auditable workflows to ensure scalability, resiliency, and minimal disruption to downstream consumers.

We can leverage a cloudformation custom resource to trigger step functions and lambdas which uses APIs/SDKs to change configurations at deploy time. This is what we call infrastructure as code model. Without custom resources, we'd need separate scripts running before/after deploy which is prone to errors and makes rollback unreliable.

![Configuration management]({{ '/assets/images/config_mgmt.png' | relative_url }})

**Operational Layer:**<br>
For operational layer, at minimum, we need to have a connect status management lambda that monitors connector and task status, auto-restart failed resources. Since Kafka Connect automatically publishes connector/task status changes to its internal status topic. We can have a Lambda with an MSK event source mapping that consumes from that topic. For the restart path, When the lambda records a FAILED state, a separate Step Function periodically scans the ConnectStatuses dynamoDB table, finds failed resources, and invokes RestartConnectResource lambda — which then calls the Connect REST API.

We need to have a connect offset monitor mechanism to calculate data replication lag. This is a two step process. The first step should be to track the connector's position. Every time a connector commits its offset (i.e. "I've read up to row ID X at timestamp Y"), Kafka Connect writes that to the offsets topic. This Lambda consumes it and stores the connector's last-known position in CdcConnectOffsets DynamoDB table. The second step is to calculate the lag. We can have a lambda to read from the dynamoDB to get the connector's committed position from step 1. Then we can query postgres directly to get the maximum id, using a simple query such as:

```sql
SELECT id FROM cdc_events WHERE id = (SELECT max(id) FROM cdc_events)
```

Then we can get the lag by calculating the difference between latest cdc event id and the committed event id: 

```java
lag = latestCdcEventId - committedCdcEventId
```
A growing lag would indicate the connector is falling behind. The number would also give a sense of how many events lapsed between the committed and the latest event. 

We also need a cdc events partition manager that creates future partitions, drop old ones on schedule because once connector read the data and send to kafka successfully the old data have no value, without partition cleaning the old data would require delete query using the logdate, which is inefficient. With partition, we can just drop the partition which is very efficient.

**Testing:**<br>
Last but not least, designing a robust E2E test pipeline is important and provides the confidence that new changes won't break or disrupt production. I will cover the E2E tests in another post.