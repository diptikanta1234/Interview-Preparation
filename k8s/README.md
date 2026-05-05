# Kubernetes Notes:-
-------------------

https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/
<img width="731" height="352" alt="image" src="https://github.com/user-attachments/assets/3c126019-3cee-4ee9-af86-7200f9c1a7d0" />
<img width="645" height="224" alt="image" src="https://github.com/user-attachments/assets/445dae9a-8153-43aa-ba58-628e02ad4300" />
<img width="1863" height="938" alt="image" src="https://github.com/user-attachments/assets/27dfd6a0-d043-40c5-b3ba-398d63ea8390" />

https://www.youtube.com/watch?v=LwC71u2DyD0&t=437s
KEDA vs HPA: Major Advantages
1. Event-Driven Scaling (Beyond CPU/Memory)
HPA only scales on CPU and memory metrics. KEDA can scale based on virtually any event source:

Message queue length (Kafka, RabbitMQ, Azure Service Bus, SQS)
Database query results
HTTP request rate
Cron schedules
Custom metrics from Prometheus, Datadog, etc.

2. Scale to Zero
KEDA can scale deployments all the way down to 0 replicas when there's no work to do, then back up when events arrive. HPA has a minimum of 1 replica (unless you use a workaround), making KEDA significantly more cost-efficient for intermittent workloads.
3. Richer Scaler Ecosystem
KEDA ships with 60+ built-in scalers (AWS, Azure, GCP, Kafka, Redis, MongoDB, etc.). HPA requires you to set up a custom metrics adapter for anything beyond CPU/RAM, which is complex and fragile.
