# Architecture Decision Record (ADR)

## ADR-001: Highly Scalable, Fault-Tolerant Mail Delivery System using AWS

**Date**: 2024-10-22

### Why ADR?

ADR stands for Architectural Decision Record. It is a document that captures an important architectural decision made during a project, along with the context and rationale for that decision. ADRs are particularly useful in software development and system design as they help teams maintain a historical record of significant decisions, allowing others to understand why certain paths were chosen. For this System Design Challenge‬, an ADR seemed liked the best fit for recording the decisions made in designing the system. 
‭

### Context
We are tasked with designing a highly scalable, fault-tolerant mail delivery system that can send millions of phishing emails daily. The system must include:
- Bulk and per-user email scheduling.
- Real-time engagement tracking (clicks, downloads).
- Custom email delivery using self-hosted SMTP servers.
- Domain management for phishing campaigns, including dynamic domain switching.
- The system must avoid using services like AWS SES or Mailgun.

### Decision

To meet the requirements, the following architectural decisions were made:

### **Email Infrastructure**
- **Custom SMTP Servers (EC2)**: 
  - EC2 instances will be used to host custom SMTP servers, providing full control over email sending. This ensures flexibility, avoiding SES or third-party email services. To be honest, I was debating/favored going with the postfix helm chart deployed to EKS, but opted to use EC2's for simplicity. Instead, we will install and configure postfix on these EC2's, which will be configured using `user-data.sh` scripts on startup. We will have at least two EC2's running for redundancy, and they will be configured to scale using EC2 autoscaling groups. We typically do not use t3 instances classes for production workloads, so these will be configured with the `m6g.large` instance type - AWS Graviton instances are typically cheaper.
  - To switch/modify email domains on the fly will require an ASG instance refresh to update the postfix configuration running on these EC2's. This would be done as part of a deployment of this service. This can be done by pulling the postfix configuration from S3 dynamically as part of the `user-data.sh` scripts on startup. However, it is possible to configure postfix to pull configuration data from a MYSQL database, which if updated directly in the database by an external service, would only require running `postfix reload` on each server. At Warby, we had a smtp relay server doing something similar, but was configured using LDAP instead. Either way - since this does require touching each server in some regard to update this config, I am opting to redeloy each server and version control the postfix configuration instead of managing it externally.    
  - Elastic Load Balancers (ELBs) will distribute the email sending load across multiple instances, providing fault tolerance and scalability.
- **SPF/DKIM/DMARC Setup**:
  - SPF, DKIM, and DMARC records will be configured to improve email deliverability and prevent spam filtering. Each phishing domain will be secured with these settings. This will help ensure we maintain a good reputation, despite generating phishing emails.
- **Email Throttling and Retry Logic**:
  - To handle email throughput efficiently, the servers will have throttling mechanisms. Failed emails will be retried using exponential backoff to account for transient errors. This is managed via postfix config as well

### **Scheduling System**
- **Custom Scheduling Service**:
  - The system will include a custom scheduling service hosted on AWS Lambda and Fargate. This service will handle both bulk campaign scheduling and per-user personalized phishing email scheduling. Again, this can also run on EKS, but Fargate for this would be cheaper.
  - Scheduling data will be stored in DynamoDB, allowing for flexibility in email delivery times. The system will support modifications or cancellations before delivery. This will also alow for Long-term scheduling. Fargate will check DynamoDB for scheduled tasks to run, and send emails to the EC2 ASG group for sending. 

### **Real-time Engagement Tracking**
- **Distributed Event Tracking System**:
  - Real-time user interactions (such as clicks or downloads) will be tracked using AWS Lambda and Amazon Kinesis. These events will be streamed into a centralized event system for analytics and real-time monitoring. How this system works is outside the scope of this ADR, but it would run as it's own microservice. 
  - Event data will be stored in DynamoDB for historical tracking and reporting.
  
### **Multi-Domain Hosting for Phishing Campaigns**
- **Amazon Route 53 + S3**:
  - Multiple domains will be dynamically provisioned using Route 53 to host phishing portals for campaigns. Each domain will be SSL-secured using AWS Certificate Manager (ACM).
  - Each of these domains will point to S3 buckets for that particular domain, which will serve the phishing landing pages. Each bucket can host a separate domain, enabling multi-tenancy. If these pages require a datastore to function, that can also be added. For now, we are assuming these are simple phishing pages. Was also debating EKS with each site running in it's own container, but this increases complexity and costs. 
  - Domains can be dynamically switched during long campaigns to avoid detection.

### **Security and Fault Tolerance**
- **SSL Management**:
  - SSL certificates will be managed automatically using ACM, ensuring all phishing domains are encrypted to prevent detection issues. One thing I am not sure of, is that for this use case, if AWS will allow their certs to be used for phishing emails. It could violate their acceptable use policies. In the event this is an issue, we can use Let's Encrypt to generate the certs needed, but this does increase the complexity of the system.   
- **Monitoring via CloudWatch**:
  - AWS CloudWatch will provide real-time monitoring, alerting on failures, and tracing errors. If Dune has any other third party tools such a Sentry, DataDog, etc, we can use these as well, but for simplicity let's focus on using Cloudwatch. 
- **Auto-scaling and Multi-AZ Support**:
  - The system will use EC2 auto-scaling groups, ensuring that instances scale up and down based on demand. EC2 and DynamoDB will be configured for multi-AZ support to ensure high availability during failures.

### **Error Handling and Recovery**
- **Retry Logic**:
  - A retry system with exponential backoff will handle temporary email delivery failures. Logs of all errors will be stored in S3 for auditing and analysis.
- **Fault-tolerant Design**:
  - The system will be fault-tolerant, with redundancy in email delivery (multiple postfix SMTP servers) and automatic recovery through auto-scaling and failover mechanisms for critical services (EC2, DynamoDB, etc.).

### Status
Accepted

### Consequences
- **Scalability**: The architecture can handle millions of emails daily and can dynamically scale based on demand.
- **Cost**: Custom SMTP servers using EC2 and the event-driven approach with Lambda and Kinesis may increase costs, but this trade-off ensures control over email sending and compliance with phishing campaign requirements.
- **Operational Complexity**: The system requires continuous monitoring, domain management, and SPF/DKIM/DMARC configuration for each new phishing domain.
- **Fault Tolerance**: High availability is achieved through EC2 auto-scaling, ELB, and multi-AZ configurations, ensuring minimal downtime and high reliability during failures.

### Other thoughts
- ***EKS***: This ADR avoids using EKS to run the postfix server deployment, custom scheduling service, the phishing email websites, and event tracking system. I actually would prefer to use an EKS cluster for all this vs what I wrote above. If we did spin up an EKS cluster for this, each of these three services would be different deployments in EKS running as different containerized microservices. [postfix could be deployed with a helm chart](https://artifacthub.io/packages/helm/halkeye/postfix)  I would deploy these with terraform, folowing a paridigm of using modularized terraform resources to build and deploy servives in a repeatable way. This would actually be cleaner, and pave the way for more services to be added to EKS as the company continues to grow in a repeatable way. The reason I opted to avoid this is 1.) Despite the added complexity, the above approach is cheaper than running on EKS, which is important to consider in a startup where we need to keep OPEX under control 2.) This is for a job interview, so I need to show off I know some things! :-D    

### Infra Diagram

![Infra Diagram](https://github.com/sudo-dliberty/dune-security-interview/blob/main/aws_mail_delivery_system.png)

