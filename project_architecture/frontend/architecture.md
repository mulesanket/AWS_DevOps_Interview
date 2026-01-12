### Deployment Architecture of Application
#### Elevator

* IRIS School Spider is a three-tier microservice application deployed on AWS across multiple Availability Zones for high availability.
* The frontend is built using React with TypeScript (Vite) and is hosted on Amazon S3, delivered globally via CloudFront for low-latency access, with AWS WAF protecting the edge.
* The backend is developed in Java using the Spring Framework, built with Maven and containerized, and deployed on Amazon EKS. External API traffic is routed through an Application Load Balancer created via Kubernetes Ingress, and the EKS worker nodes run in private subnets across three AZs for security and resilience.
* The data layer uses Amazon Aurora PostgreSQL in a multi-AZ setup with a primary writer and read replicas, along with automated backups.
* From a security perspective, we use private subnets, security groups, AWS WAF, IAM Roles for Service Accounts, and AWS Secrets Manager. We also use VPC endpoints for services like S3, ECR, and Secrets Manager so AWS service traffic stays within the VPC.
* For scalability, we use Kubernetes Horizontal Pod Autoscaling for pods and Cluster Autoscaler for worker nodes, allowing the system to scale automatically based on traffic.
* CI/CD runs from an isolated build account using TeamCity with SonarQube and Snyk quality gates, and monitoring and alerting are handled using CloudWatch and Datadog.