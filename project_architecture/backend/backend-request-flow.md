### Backend: step-by-step flow  
### Step 1: ALB (Ingress) receives request
- Backend traffic for IRIS School Spider is handled by an ALB (Kubernetes Ingress) into an Amazon EKS cluster.  
- when the frontend calls backend microservice, DNS (https://api.schoolspider.co.uk/) resolves to AWS Load Balancer URL and ALB acts as the Layer-7 entry point; it does TLS termination (optionally), host/path routing and forwards HTTP to the cluster.  
- We use path-based routing to map paths to microservices (school, parent, pupil, payments, fees), and those same paths are defined in our Kubernetes Ingress resources.
- Security:  
- TLS termination at ALB (ACM certificate) — ensures encryption in transit from client → ALB.  
- ALB listener enforces HTTPS and redirects HTTP → HTTPS.  
- ALB security group allows only required inbound (CloudFront IPs or public) and only the health check port from the ALB target group.  
- Scaling/HA:  
- ALB is fully managed and auto-scales to handle varying traffic.  
- ALB is regional and multi-AZ — target groups can have targets in multiple AZs so traffic is routed to healthy targets across AZs.  
- ALB health checks are aligned with our pod readiness endpoint so only ready pods receive traffic; unhealthy targets are removed automatically. We also use target deregistration delay + pod preStop hooks to allow graceful draining of in-flight requests during scale down or deploys.    

### Step 2: ALB → Kubernetes Ingress Controller (AWS LB Controller)  
- The AWS LB Controller installed in cluster continuously watches Ingress resource's rules and configuration changes.  
- The controller configures/creates ALB and target groups using targetType=IP so pod IPs are direct targets, and keeps ALB in sync with the cluster.  
- Security:  
- Controller runs with IRSA (IAM Role for Service Accounts) using least-privilege permissions to create/configure ALBs and related resources.  
- The controller’s IAM role is tightly scoped (create/modify ALB, target groups, SGs, tags) — not full admin.  
- Security groups and network policies are configured so only the ALB can reach the required service ports on pods. and The controller does not expose credentials — it uses the service account IRSA flow so no secrets are stored in the cluster.  
- Scaling/HA:  
- The controller is deployed as a replicated Deployment (>=2 replicas) across nodes/AZs so controller pod failure doesn’t interrupt reconciliation.  
- The controller is stateless — any replica can reconcile state; Kubernetes schedules them across AZs for resilience.  
- We also have implemented HA for pods and as HPA scales pods up/down the controller updates ALB target groups dynamically so routing remains accurate.  

### Step 3 — Kubernetes Service / Pod endpoints  
- At this stage the ALB forwards traffic directly to pod IPs (targetType=ip); the AWS LB Controller keeps the ALB target group in sync with pod IPs while Kubernetes Services and EndpointSlices continue to provide internal service discovery and load-balancing for cluster-internal traffic.  
- The AWS LB Controller creates a Target Group and a TargetGroupBinding that registers pod IPs (and container ports) as targets in the ALB target group.  
- ALB → pod: incoming requests from the ALB are routed directly to the pod IP:port registered in the target group.  
- Internally, the Kubernetes Service (ClusterIP) still exists for internal clients — it uses EndpointSlices that list the same pod IPs for in-cluster load balancing. 
- When HPA adds/removes pod replicas, the controller registers/deregisters pod IPs in the ALB target group dynamically so ALB always points to healthy pods.  
- EndpointSlices update immediately for in-cluster discovery; internal traffic continues to work via Service IP. 

### Step 4  
- Inside each pod we run NGINX as a lightweight reverse proxy in front of the Spring Boot process. NGINX handles buffering, timeouts and graceful draining while Spring executes business logic and talks to Aurora through a connection pool. We use startup/readiness/liveness probes (Actuator endpoints) so ALB only routes to ready pods; pod preStop hooks and terminationGracePeriod + ALB deregistration ensure graceful shutdown. Pods declare resource requests/limits, HPA scales replicas, and IRSA + Secrets Manager supply credentials at runtime. Observability is via logs, metrics and distributed traces so we can quickly detect and remediate issues.  

### Step 5: Amazon Aurora PostgreSQL  
Application pods connect to a multi-AZ Amazon Aurora PostgreSQL cluster for persistence.  
- we use connection pooling, secure networking and managed Aurora features (read-replicas, automated failover and snapshots) to ensure performance and availability.  
- The Spring app (inside the pod) uses a DB connection pool (HikariCP) to open and reuse connections to the Aurora cluster writer/reader endpoints.  
- Reads can go to the reader endpoint (read replicas) and writes go to the cluster writer endpoint (primary).  
- Connections use TLS/SSL and authenticate with credentials retrieved securely (Secrets Manager) or via IAM auth if enabled.  

- Security:   
- Network isolation: Aurora runs in private DB subnets and only allows inbound from EKS node/pod security groups.  
- Encryption: Data encrypted at rest (KMS) and TLS enforced for in-transit traffic.  
- Credentials: DB credentials are stored/rotated in AWS Secrets Manager and injected at runtime (CSI driver or SDK). Optionally use IAM DB authentication.  
- Least privilege: DB user permissions follow principle of least privilege; production-only accounts separate from CI/build accounts.  
- Optional: Use RDS Proxy to pool/authenticate connections centrally and reduce DB connection churn.

- Scaling & performance:  
- Read scaling: Add read replicas and use the reader endpoint to distribute read traffic.  
- Write scaling: Scale vertically (instance class) or horizontally by sharding/partitioning at the application level; Aurora Serverless or higher instance classes as options.  
- Connection pooling: HikariCP per pod sized so total connections across pods ≤ DB max connections. (Example: if DB max = 500 and you plan 10 pods, size maxPool ~ 40–45 per pod.)  
- RDS Proxy: Consider RDS Proxy to multiplex many app connections into fewer DB connections and prevent connection storms during scale-up.  

- High availability & failover:  
- Multi-AZ: Aurora stores data across AZs and automatically fails over to a healthy replica if the primary fails.  
- Fast failover: Promotion of a replica is automatic (managed by Aurora), minimizing downtime; we rely on the managed failover behavior.  
- Backups & PITR: Automated snapshots and point-in-time recovery (PITR) are enabled; snapshots can be copied cross-region for DR.  
- Maintenance: Apply minor updates in maintenance windows and test major upgrades in staging first.  

- Monitoring & alerts:  
- Monitor CloudWatch metrics: CPUUtilization, DatabaseConnections, FreeableMemory, ReplicaLag, Deadlocks, DiskQueueDepth, VolumeBytesUsed.  
- Enable Performance Insights for slow queries and tuning.  
- Alerts: DB connections >80% of max, replica lag > threshold, sustained CPU > 70%, or failed automated backups.  

- Troubleshooting & runbook bullets:

- If DB connections exhausted: Reduce per-pod pool, add read replicas or scale instance, enable RDS Proxy.  
- If replica lag increases: Reduce heavy read queries, scale replica instance class, check long-running queries.  
- If failover occurred: Verify app reconnect logic (JDBC retries), check logs and promote/copy snapshots if cross-region DR needed.  
- If slow queries: Use Performance Insights + slow query log to optimize indexes/queries.  

- Pods use a HikariCP connection pool to talk to a multi-AZ Amazon Aurora PostgreSQL cluster. Writes go to the cluster writer endpoint and reads use the reader endpoint with read replicas for scaling. Aurora runs in private DB subnets with security groups that only allow EKS access; credentials are pulled from Secrets Manager (or IAM auth) and all traffic is TLS encrypted. We size per-pod DB pools so total connections don’t exceed DB limits, use read replicas (and optionally RDS Proxy) for scale, and rely on Aurora’s automatic multi-AZ failover plus automated snapshots/PITR for availability and recovery. Monitoring (CloudWatch + Performance Insights) and alerts for connections, replica lag and CPU help us operate and tune the DB.