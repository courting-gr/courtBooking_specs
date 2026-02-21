# Requirements Document

## Introduction

Phase 1 establishes the foundational infrastructure for the Court Booking Platform on DigitalOcean. This phase delivers the complete infrastructure-as-code foundation using Terraform, CI/CD pipelines via GitHub Actions, Kubernetes cluster setup, managed database and cache services, and local development environment configuration. Upon completion, both Spring Boot services (Platform Service and Transaction Service) will be running on DOKS (empty but healthy), CI/CD pipelines will be green, and the local development environment will be fully operational.

## Glossary

- **DOKS**: DigitalOcean Kubernetes Service — managed Kubernetes offering
- **Terraform**: Infrastructure as Code tool for provisioning cloud resources
- **VPC**: Virtual Private Cloud — isolated network for resources
- **Spaces**: DigitalOcean's S3-compatible object storage service
- **DOCR**: DigitalOcean Container Registry — private Docker image storage
- **PostGIS**: PostgreSQL extension for geospatial data
- **Flyway**: Database migration tool for version-controlled schema changes
- **Sealed_Secrets**: Kubernetes-native secret management using encrypted secrets in Git
- **NGINX_Ingress**: Kubernetes ingress controller for HTTP/HTTPS routing
- **Helm**: Kubernetes package manager for deploying applications
- **Kustomize**: Kubernetes configuration management tool for environment overlays
- **Shared_Cluster**: Single DOKS cluster hosting dev, test, and staging namespaces
- **Production_Cluster**: Dedicated DOKS cluster for production workloads
- **Terraform_Module**: Reusable Terraform code encapsulating related resources
- **Composition**: Terraform configuration combining modules for a specific environment
- **Remote_State**: Terraform state stored in DigitalOcean Spaces for team collaboration
- **GitHub_Environment**: GitHub feature for deployment protection rules and secrets

## Requirements

### Requirement 1: Terraform Module Structure

**User Story:** As a DevOps engineer, I want a well-organized Terraform module structure, so that infrastructure code is reusable, maintainable, and follows industry best practices.

#### Acceptance Criteria

1. THE Infrastructure_Repository SHALL follow the monorepo layout with `modules/`, `environments/`, `kubernetes/`, and `scripts/` directories at the root level
2. THE Infrastructure_Repository SHALL contain reusable resource modules in `modules/` for: `vpc`, `doks-cluster`, `managed-postgres`, `managed-redis`, `spaces`, `container-registry`, `dns`, and `project`
3. WHEN a Terraform module is created, THE Module SHALL contain exactly these files: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, and `README.md`
4. THE `main.tf` file SHALL contain only resource blocks, data sources, and locals — never variable or output declarations
5. THE `variables.tf` file SHALL contain all variable blocks with description, type, and optional default in that order
6. THE `outputs.tf` file SHALL contain all output blocks with description and value
7. THE `versions.tf` file SHALL contain the `terraform { required_providers {} }` block with version constraints
8. WHEN naming Terraform identifiers (resources, variables, outputs), THE Infrastructure_Code SHALL use underscores (`_`) as word separators
9. WHEN naming cloud resources visible in DigitalOcean console, THE Infrastructure_Code SHALL use dashes (`-`) as word separators with pattern `{project}-{environment}-{purpose}`
10. WHEN a module contains only one resource of a type, THE Resource SHALL be named `this` (e.g., `digitalocean_vpc.this`)
11. WHEN a resource type is repeated in the name, THE Infrastructure_Code SHALL omit the redundant type (e.g., `digitalocean_database_cluster.primary` not `primary_database_cluster`)

### Requirement 2: Conditional Resource Creation Pattern

**User Story:** As a DevOps engineer, I want modules to support conditional creation, so that I can enable or disable entire modules without code changes.

#### Acceptance Criteria

1. EVERY Terraform module SHALL include a `create` variable of type `bool` with default value `true`
2. WHEN `create` is `false`, THE Module SHALL create zero resources by using `count = var.create ? 1 : 0` on all resources
3. WHEN outputting values from conditionally created resources, THE Module SHALL use `try()` function for safe access (e.g., `try(digitalocean_kubernetes_cluster.this[0].id, null)`)
4. THE Module outputs SHALL return `null` or empty string when resources are not created, never causing Terraform errors

### Requirement 3: VPC Module

**User Story:** As a DevOps engineer, I want isolated VPC networks for each environment, so that resources are securely segmented.

#### Acceptance Criteria

1. THE VPC_Module SHALL create a DigitalOcean VPC with configurable name, region, description, and IP range
2. THE Shared_Environment SHALL use IP range `10.10.0.0/16` for its VPC
3. THE Production_Environment SHALL use IP range `10.20.0.0/16` for its VPC
4. THE VPC_Module SHALL output the VPC ID for use by other modules (DOKS, PostgreSQL, Redis)
5. THE VPC_Module SHALL apply default tags: `managed-by:terraform`, `project:court-booking`, `environment:{env}`

### Requirement 4: DOKS Cluster Module

**User Story:** As a DevOps engineer, I want managed Kubernetes clusters with auto-scaling, so that the platform can handle variable workloads efficiently.

#### Acceptance Criteria

1. THE DOKS_Module SHALL create a DigitalOcean Kubernetes cluster with configurable name, region, version, and VPC association
2. THE DOKS_Module SHALL support a default node pool with configurable size, auto-scaling (min/max nodes), and node count
3. THE DOKS_Module SHALL support additional node pools via a `for_each` pattern with configurable taints and labels
4. THE Shared_Cluster SHALL be configured with: 2-4 nodes auto-scaling, `s-4vcpu-8gb` droplet size, HA disabled
5. THE Production_Cluster SHALL be configured with: 3-6 nodes auto-scaling, `s-4vcpu-8gb` droplet size, HA enabled
6. THE Production_Cluster SHALL include a dedicated observability node pool with `s-2vcpu-4gb` size, 1-2 nodes, and `dedicated=observability:NoSchedule` taint
7. THE DOKS_Module SHALL enable container registry integration for pulling images from DOCR
8. THE DOKS_Module SHALL configure maintenance windows (Sunday 03:00-04:00 UTC for production, 04:00-05:00 for shared)
9. THE DOKS_Module SHALL enable auto-upgrade and surge-upgrade for production clusters
10. THE DOKS_Module SHALL use `lifecycle { ignore_changes = [version] }` to prevent drift on Kubernetes version
11. THE DOKS_Module SHALL output cluster ID, endpoint, kubeconfig, and cluster CA certificate

### Requirement 5: Managed PostgreSQL Module

**User Story:** As a DevOps engineer, I want managed PostgreSQL with PostGIS support, so that the platform has reliable geospatial database capabilities.

#### Acceptance Criteria

1. THE PostgreSQL_Module SHALL create a DigitalOcean Managed PostgreSQL cluster with engine `pg` version 16
2. THE PostgreSQL_Module SHALL configure the cluster within the environment's VPC for private networking
3. THE Shared_Environment SHALL use `db-s-1vcpu-2gb` size with 1 node and no read replica
4. THE Production_Environment SHALL use `db-s-1vcpu-2gb` size with 1 node and a read replica for analytics queries
5. THE PostgreSQL_Module SHALL create separate databases: `platform` and `transaction` for schema isolation
6. THE PostgreSQL_Module SHALL create separate database users: `platform_service` and `transaction_service`
7. THE PostgreSQL_Module SHALL configure database firewall rules to allow connections only from the DOKS cluster
8. THE PostgreSQL_Module SHALL configure maintenance windows during low-traffic hours
9. THE PostgreSQL_Module SHALL output connection host, port, database names, and user credentials (marked sensitive)
10. WHEN the read replica is created, THE PostgreSQL_Module SHALL output the replica's connection details separately

### Requirement 6: Managed Redis Module

**User Story:** As a DevOps engineer, I want managed Redis for caching and pub/sub, so that the platform has reliable in-memory data storage.

#### Acceptance Criteria

1. THE Redis_Module SHALL create a DigitalOcean Managed Redis cluster with version 7
2. THE Redis_Module SHALL configure the cluster within the environment's VPC for private networking
3. THE Redis_Module SHALL use `db-s-1vcpu-2gb` size (2GB RAM) for both shared and production environments
4. THE Redis_Module SHALL configure eviction policy as `allkeys-lru` for cache use cases
5. THE Redis_Module SHALL configure database firewall rules to allow connections only from the DOKS cluster
6. THE Redis_Module SHALL output connection host, port, and password (marked sensitive)

### Requirement 7: Spaces Module (Object Storage)

**User Story:** As a DevOps engineer, I want S3-compatible object storage, so that the platform can store court images, assets, and Terraform state.

#### Acceptance Criteria

1. THE Spaces_Module SHALL create a DigitalOcean Spaces bucket with configurable name, region, and ACL
2. THE Spaces_Module SHALL support CORS configuration for web access to court images
3. THE Spaces_Module SHALL support lifecycle rules for automatic cleanup of temporary files
4. THE Spaces_Module SHALL support optional CDN enablement for public asset delivery
5. THE Infrastructure_Repository SHALL use a dedicated Spaces bucket `court-booking-terraform-state` for Terraform remote state
6. THE Spaces_Module SHALL output bucket name, endpoint, and CDN endpoint (if enabled)

### Requirement 8: Container Registry Module

**User Story:** As a DevOps engineer, I want a private container registry, so that Docker images are securely stored and accessible to DOKS clusters.

#### Acceptance Criteria

1. THE Registry_Module SHALL create a DigitalOcean Container Registry with `basic` subscription tier
2. THE Registry_Module SHALL be created in the `fra1` region
3. THE Registry_Module SHALL output registry endpoint and Docker credentials for CI/CD integration
4. THE DOKS clusters SHALL have registry integration enabled for seamless image pulling

### Requirement 9: DNS Module

**User Story:** As a DevOps engineer, I want DNS management through Terraform, so that domain records are version-controlled and consistent.

#### Acceptance Criteria

1. THE DNS_Module SHALL manage DigitalOcean domain and DNS records
2. THE DNS_Module SHALL support A records pointing to load balancer IPs
3. THE DNS_Module SHALL support CNAME records for environment-specific subdomains
4. THE DNS_Module SHALL output domain name and record IDs

### Requirement 10: Load Balancer Configuration

**User Story:** As a DevOps engineer, I want load balancers with SSL termination, so that traffic is securely distributed to Kubernetes services.

#### Acceptance Criteria

1. THE Load_Balancer SHALL be created automatically by the NGINX Ingress Controller via Kubernetes service annotations
2. THE Load_Balancer SHALL use DigitalOcean's automatic SSL/TLS with Let's Encrypt certificates
3. THE Load_Balancer SHALL be named using the pattern `{project}-{environment}-lb`
4. THE Load_Balancer SHALL forward HTTPS traffic (port 443) to the ingress controller

### Requirement 11: Environment Compositions

**User Story:** As a DevOps engineer, I want environment-specific Terraform compositions, so that shared and production environments are independently managed.

#### Acceptance Criteria

1. THE Infrastructure_Repository SHALL contain `environments/shared/` composition for dev, test, and staging namespaces
2. THE Infrastructure_Repository SHALL contain `environments/production/` composition for the production namespace
3. EACH composition SHALL contain: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `backend.tf`, and `terraform.tfvars`
4. THE `backend.tf` SHALL configure DigitalOcean Spaces as the S3-compatible backend with appropriate state file keys
5. THE Shared_Composition SHALL use state key `shared/terraform.tfstate`
6. THE Production_Composition SHALL use state key `production/terraform.tfstate`
7. THE compositions SHALL call modules with environment-specific parameters (HA, node counts, replica settings)

### Requirement 12: Terraform State Management

**User Story:** As a DevOps engineer, I want remote state storage with locking, so that team members can safely collaborate on infrastructure changes.

#### Acceptance Criteria

1. THE Terraform_State SHALL be stored in DigitalOcean Spaces bucket `court-booking-terraform-state`
2. THE Backend_Configuration SHALL use S3-compatible endpoint `https://fra1.digitaloceanspaces.com`
3. THE Backend_Configuration SHALL include `skip_credentials_validation`, `skip_metadata_api_check`, `skip_requesting_account_id`, and `skip_s3_checksum` settings for DigitalOcean compatibility
4. WHEN cross-environment data access is needed, THE Composition SHALL use `terraform_remote_state` data source
5. THE State files SHALL be isolated: shared and production environments SHALL NOT share state

### Requirement 13: GitHub Actions CI/CD Pipeline for Terraform

**User Story:** As a DevOps engineer, I want automated Terraform pipelines, so that infrastructure changes are reviewed, tested, and deployed safely.

#### Acceptance Criteria

1. THE Infrastructure_Repository SHALL contain `.github/workflows/terraform-pr.yml` for PR workflows
2. THE Infrastructure_Repository SHALL contain `.github/workflows/terraform-deploy.yml` for merge workflows
3. WHEN a PR is opened or updated, THE PR_Workflow SHALL run `terraform plan` for both shared and production environments
4. WHEN a PR is opened, THE PR_Workflow SHALL post plan output as a PR comment (truncated to 60KB if needed)
5. WHEN plans complete, THE PR_Workflow SHALL require manual approval via GitHub Environment `dev` before deploying to dev namespace
6. WHEN a PR is merged to main, THE Deploy_Workflow SHALL automatically deploy to test environment
7. AFTER test deployment, THE Deploy_Workflow SHALL require manual approval via GitHub Environment `staging` before deploying to staging
8. AFTER staging deployment, THE Deploy_Workflow SHALL require manual approval via GitHub Environment `production` before deploying to production
9. THE Workflows SHALL save plan artifacts and reuse them during apply to ensure reviewed changes are applied
10. THE Workflows SHALL re-plan before each deployment to catch state drift
11. THE Workflows SHALL use Terraform version 1.9.0 (pinned via `TF_VERSION` environment variable)

### Requirement 14: GitHub Environments and Secrets

**User Story:** As a DevOps engineer, I want GitHub environments with protection rules, so that deployments require appropriate approvals.

#### Acceptance Criteria

1. THE GitHub_Repository SHALL have four environments configured: `dev`, `test`, `staging`, `production`
2. THE `dev` Environment SHALL have optional required reviewers
3. THE `test` Environment SHALL have no protection rules (auto-deploy)
4. THE `staging` Environment SHALL require at least one reviewer approval
5. THE `production` Environment SHALL require at least one reviewer approval AND restrict deployments to `main` branch only
6. THE GitHub_Repository SHALL store these secrets: `DO_TOKEN`, `SPACES_ACCESS_ID`, `SPACES_SECRET_KEY`
7. THE Workflows SHALL pass secrets as Terraform variables via `TF_VAR_` environment variables

### Requirement 15: Kubernetes Namespace Setup

**User Story:** As a DevOps engineer, I want Kubernetes namespaces for environment isolation, so that dev, test, staging, and production workloads are separated.

#### Acceptance Criteria

1. THE Shared_Cluster SHALL have namespaces: `dev`, `test`, `staging`
2. THE Production_Cluster SHALL have namespace: `production`
3. EACH namespace SHALL have labels: `environment`, `managed_by:terraform`, `project:court-booking`
4. THE `staging` namespace SHALL have label `istio_inject:enabled` for service mesh validation
5. THE `dev` and `test` namespaces SHALL have label `istio_inject:disabled` to conserve resources
6. THE Kubernetes manifests SHALL be organized using Kustomize with `base/` and `overlays/` structure

### Requirement 16: NGINX Ingress Controller

**User Story:** As a DevOps engineer, I want an ingress controller for HTTP routing, so that services are accessible via path-based routing with SSL.

#### Acceptance Criteria

1. THE Infrastructure SHALL deploy NGINX Ingress Controller via Helm chart `ingress-nginx`
2. THE Ingress_Controller SHALL be deployed to namespace `ingress-nginx`
3. THE Ingress_Controller SHALL configure DigitalOcean Load Balancer annotations for automatic provisioning
4. THE Ingress_Controller SHALL support path-based routing rules for Platform Service and Transaction Service
5. THE Ingress_Controller SHALL terminate SSL using Let's Encrypt certificates via cert-manager
6. THE Helm values SHALL be stored in `kubernetes/helm-values/nginx-ingress-values.yaml`

### Requirement 17: Cert-Manager for SSL Certificates

**User Story:** As a DevOps engineer, I want automated SSL certificate management, so that HTTPS is enabled without manual certificate handling.

#### Acceptance Criteria

1. THE Infrastructure SHALL deploy cert-manager via Helm chart
2. THE cert-manager SHALL be configured with Let's Encrypt ClusterIssuer for automatic certificate provisioning
3. THE cert-manager SHALL support both staging (for testing) and production Let's Encrypt endpoints
4. THE Ingress resources SHALL reference cert-manager annotations for automatic certificate creation

### Requirement 18: Observability Stack (Production)

**User Story:** As a DevOps engineer, I want a complete observability stack, so that the platform can be monitored, traced, and debugged effectively.

#### Acceptance Criteria

1. THE Production_Cluster SHALL deploy Prometheus via `kube-prometheus-stack` Helm chart for metrics collection
2. THE Production_Cluster SHALL deploy Grafana for dashboards and alerting
3. THE Production_Cluster SHALL deploy Jaeger for distributed tracing
4. THE Production_Cluster SHALL deploy Loki for log aggregation
5. THE Observability_Stack SHALL be deployed to the dedicated observability node pool using node selectors and tolerations
6. THE Helm values SHALL be stored in `kubernetes/helm-values/` directory
7. THE Shared_Cluster SHALL use DigitalOcean built-in monitoring for MVP to conserve resources
8. IF Grafana Cloud free tier is used for MVP, THE Infrastructure SHALL configure remote write for Prometheus metrics

### Requirement 19: Sealed Secrets for Secret Management

**User Story:** As a DevOps engineer, I want encrypted secrets in Git, so that sensitive configuration can be version-controlled safely.

#### Acceptance Criteria

1. THE Infrastructure SHALL deploy Sealed Secrets controller via Helm chart
2. THE Sealed_Secrets SHALL encrypt secrets using the cluster's public key
3. THE Encrypted secrets SHALL be stored in Git alongside Kubernetes manifests
4. THE Sealed_Secrets controller SHALL decrypt secrets at runtime within the cluster
5. THE Infrastructure SHALL document the migration path to HashiCorp Vault for future operational maturity

### Requirement 20: Database Firewall Rules

**User Story:** As a DevOps engineer, I want database access restricted to the Kubernetes cluster, so that databases are not exposed to the public internet.

#### Acceptance Criteria

1. THE PostgreSQL_Firewall SHALL allow connections only from the DOKS cluster (using `type = "k8s"` and cluster ID)
2. THE Redis_Firewall SHALL allow connections only from the DOKS cluster
3. THE Firewall_Rules SHALL be applied via `digitalocean_database_firewall` resources
4. THE Databases SHALL NOT have any public access rules

### Requirement 21: Resource Tagging Strategy

**User Story:** As a DevOps engineer, I want consistent resource tagging, so that resources are easily identifiable and cost-trackable.

#### Acceptance Criteria

1. ALL DigitalOcean resources SHALL have these default tags: `managed-by:terraform`, `project:court-booking`, `environment:{env}`
2. THE Modules SHALL accept additional tags via a `tags` variable of type `list(string)`
3. THE Modules SHALL merge default tags with custom tags using `distinct(concat(local.default_tags, var.tags))`
4. THE Tags SHALL use colon (`:`) as key-value separator per DigitalOcean convention

### Requirement 22: Docker Compose for Local Development

**User Story:** As a developer, I want a local development environment, so that I can develop and test without cloud dependencies.

#### Acceptance Criteria

1. THE Infrastructure_Repository SHALL contain a `docker-compose.yml` file at the repository root for local development (note: this is the `court-booking-infrastructure` repo, separate from the workspace-level docker-compose.yml)
2. THE Docker_Compose SHALL include PostgreSQL with PostGIS extension (image: `postgis/postgis:15-3.3`)
3. THE Docker_Compose SHALL include Redis (image: `redis:7-alpine`)
4. THE Docker_Compose SHALL include Kafka in KRaft mode (image: `confluentinc/cp-kafka:7.6.0`)
5. THE PostgreSQL service SHALL expose port 5432 with database `courtbooking`, user `dev`, password `dev`
6. THE Redis service SHALL expose port 6379
7. THE Kafka service SHALL expose port 9092 with single-node configuration
8. THE Docker_Compose SHALL use named volumes for data persistence

### ~~Requirement 23: Spring Boot Project Scaffolding~~ — MOVED to Phase 1b (Service Scaffolding spec)

> **Note:** Spring Boot scaffolding, Flyway migrations, shared common library, and application CI/CD pipelines target the `court-booking-platform-service`, `court-booking-transaction-service`, and `court-booking-common` repositories respectively. They are out of scope for this infrastructure-focused spec and will be covered in a separate "phase-1-service-scaffolding" spec.

### ~~Requirement 24: Database Schema Migrations~~ — MOVED to Phase 1b (Service Scaffolding spec)

> See note above.

### ~~Requirement 25: Shared Common Library~~ — MOVED to Phase 1b (Service Scaffolding spec)

> See note above.

### Requirement 26: Kubernetes Deployment Manifests

**User Story:** As a DevOps engineer, I want Kubernetes manifests for service deployment, so that services can be deployed consistently across environments.

#### Acceptance Criteria

1. THE Kubernetes manifests SHALL be organized using Kustomize with `base/` containing shared resources
2. THE `overlays/` directory SHALL contain environment-specific patches for `dev`, `test`, `staging`, `production`
3. THE Deployment manifests SHALL configure resource requests and limits appropriate for each environment
4. THE Deployment manifests SHALL configure liveness, readiness, and startup probes pointing to Actuator endpoints
5. THE Deployment manifests SHALL configure environment variables for database, Redis, and Kafka connections
6. THE Service manifests SHALL expose services within the cluster on appropriate ports
7. THE Ingress manifests SHALL configure path-based routing to Platform Service and Transaction Service

### ~~Requirement 27: CI/CD Pipeline for Application Services~~ — MOVED to Phase 1b (Service Scaffolding spec)

> See note on Requirements 23-25 above.

### Requirement 28: Backup and Recovery Configuration

**User Story:** As a DevOps engineer, I want automated backups with tested recovery, so that data can be restored in case of failure.

#### Acceptance Criteria

1. THE PostgreSQL_Cluster SHALL have automatic daily backups enabled with 7-day retention
2. THE PostgreSQL_Cluster SHALL have point-in-time recovery (PITR) enabled with 7-day window
3. THE Infrastructure SHALL target Recovery Point Objective (RPO) of 1 hour via WAL archiving
4. THE Infrastructure SHALL target Recovery Time Objective (RTO) of 30 minutes via managed failover
5. THE `scripts/` directory SHALL contain `backup-verify.sh` for quarterly backup restoration testing
6. THE Backup_Verification script SHALL restore to a temporary instance and validate data integrity

### Requirement 29: Maintenance Windows

**User Story:** As a DevOps engineer, I want scheduled maintenance windows, so that updates occur during low-traffic periods.

#### Acceptance Criteria

1. THE DOKS_Clusters SHALL have maintenance windows configured for Sunday early morning (UTC)
2. THE PostgreSQL_Clusters SHALL have maintenance windows configured for Sunday early morning (UTC)
3. THE Redis_Clusters SHALL have maintenance windows configured for Sunday early morning (UTC)
4. THE Production maintenance window SHALL be 03:00-04:00 UTC
5. THE Shared maintenance window SHALL be 04:00-05:00 UTC (offset to avoid simultaneous updates)

### Requirement 30: Operational Scripts

**User Story:** As a DevOps engineer, I want operational scripts for common tasks, so that maintenance operations are standardized and repeatable.

#### Acceptance Criteria

1. THE `scripts/` directory SHALL contain `backup-verify.sh` for backup restoration testing
2. THE `scripts/` directory SHALL contain `secret-rotation.sh` for rotating database passwords and API keys
3. THE `scripts/` directory SHALL contain `cluster-maintenance.sh` for DOKS cluster operations
4. THE Scripts SHALL be documented with usage instructions and prerequisites
5. THE Scripts SHALL use `doctl` CLI for DigitalOcean API operations

### Requirement 31: External Services Documentation

**User Story:** As a DevOps engineer, I want documentation for external service setup, so that manual configuration steps are clearly defined.

#### Acceptance Criteria

1. THE Documentation SHALL include setup instructions for Redpanda Serverless (Kafka-compatible event streaming)
2. THE Documentation SHALL include setup instructions for Stripe account and API keys
3. THE Documentation SHALL include setup instructions for SendGrid email service
4. THE Documentation SHALL include setup instructions for Firebase Cloud Messaging
5. THE Documentation SHALL include setup instructions for OpenWeatherMap API
6. THE Documentation SHALL include setup instructions for OAuth providers (Google, Facebook, Apple)
7. THE Documentation SHALL specify which credentials need to be added to GitHub Secrets or Sealed Secrets

### Requirement 32: DigitalOcean Project Organization

**User Story:** As a DevOps engineer, I want resources grouped in a DigitalOcean project, so that resources are organized and cost-trackable.

#### Acceptance Criteria

1. THE Infrastructure SHALL create a DigitalOcean Project named `court-booking`
2. ALL created resources SHALL be associated with the `court-booking` project
3. THE Project module SHALL output the project ID for use by other modules

### Requirement 33: Terraform Version and Provider Constraints

**User Story:** As a DevOps engineer, I want pinned Terraform and provider versions, so that infrastructure builds are reproducible.

#### Acceptance Criteria

1. THE Infrastructure SHALL require Terraform version `>= 1.5.0`
2. THE Infrastructure SHALL use DigitalOcean provider version `~> 2.75`
3. THE Infrastructure SHALL use Kubernetes provider version `~> 2.35`
4. THE Infrastructure SHALL use Helm provider version `~> 2.17`
5. THE Repository SHALL contain `.terraform-version` file for tfenv version pinning
6. THE Repository SHALL contain `.tflint.hcl` for linting configuration

### Requirement 34: Two-Stage Kubernetes Apply Pattern

**User Story:** As a DevOps engineer, I want infrastructure and Kubernetes resources applied separately, so that cluster creation completes before resource deployment.

#### Acceptance Criteria

1. THE Infrastructure_Apply (Stage 1) SHALL create DOKS clusters, databases, VPC, and other DigitalOcean resources
2. THE Kubernetes_Apply (Stage 2) SHALL deploy namespaces, ingress controller, cert-manager, and observability stack
3. THE Kubernetes provider SHALL use `data.digitalocean_kubernetes_cluster` to reference existing clusters
4. THE Helm provider SHALL use the same data source for cluster authentication
5. THE Two-stage pattern SHALL prevent circular dependencies between cluster creation and Kubernetes resource deployment

### Requirement 35: Cost Optimization for Non-Production

**User Story:** As a DevOps engineer, I want cost-optimized non-production environments, so that infrastructure costs are minimized while maintaining functionality.

#### Acceptance Criteria

1. THE Shared_Cluster SHALL disable HA to reduce control plane costs
2. THE Shared_Cluster SHALL use smaller node counts (2-4) compared to production (3-6)
3. THE Shared_PostgreSQL SHALL not create a read replica
4. THE Shared_Environment SHALL use DigitalOcean built-in monitoring instead of full observability stack
5. THE Shared_Environment SHALL disable Istio in dev and test namespaces (enabled only in staging)
6. THE Estimated monthly cost for shared environment SHALL be approximately $130-230
7. THE Estimated monthly cost for production environment SHALL be approximately $200-350

