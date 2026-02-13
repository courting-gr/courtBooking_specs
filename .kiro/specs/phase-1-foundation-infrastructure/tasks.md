# Implementation Plan: Phase 1 Foundation Infrastructure

## Overview

This implementation plan breaks down the Phase 1 infrastructure setup into discrete, executable tasks. Each task builds on previous tasks and ends with integrated, working infrastructure. The focus is on Terraform modules, CI/CD pipelines, Kubernetes setup, and local development environment.

## Tasks

- [x] 1. Initialize repository structure and tooling
  - [x] 1.1 Create infrastructure repository directory structure
    - Create `modules/`, `environments/`, `kubernetes/`, `scripts/`, `.github/workflows/` directories
    - Create `.terraform-version` file with version `1.9.0`
    - Create `.tflint.hcl` with recommended rules from steering file
    - Create `.pre-commit-config.yaml` with terraform hooks
    - _Requirements: 1.1, 1.2, 33.5, 33.6_
  
  - [x] 1.2 Create Terraform backend configuration
    - Create `environments/shared/backend.tf` with Spaces S3-compatible backend
    - Create `environments/production/backend.tf` with separate state key
    - Configure `skip_credentials_validation`, `skip_metadata_api_check`, `skip_requesting_account_id`, `skip_s3_checksum`
    - _Requirements: 12.1, 12.2, 12.3, 12.5_

  - [x] 1.3 Create versions.tf files for environments
    - Create `environments/shared/versions.tf` with provider constraints
    - Create `environments/production/versions.tf` with provider constraints
    - Pin Terraform `>= 1.5.0`, DigitalOcean `~> 2.75`, Kubernetes `~> 2.35`, Helm `~> 2.17`
    - _Requirements: 33.1, 33.2, 33.3, 33.4_

- [x] 2. Implement VPC module
  - [x] 2.1 Create VPC module structure
    - Create `modules/vpc/main.tf` with `digitalocean_vpc` resource
    - Create `modules/vpc/variables.tf` with `create`, `vpc_name`, `region`, `ip_range`, `description`, `tags`
    - Create `modules/vpc/outputs.tf` with `vpc_id`, `vpc_urn` using `try()` for safe access
    - Create `modules/vpc/versions.tf` with provider requirements
    - Create `modules/vpc/README.md` with usage examples
    - Implement default tags: `managed-by:terraform`, `project:court-booking`, `environment:{env}`
    - _Requirements: 1.3, 1.4, 1.5, 1.6, 1.7, 2.1, 2.2, 2.3, 2.4, 3.1, 3.4, 3.5, 21.1, 21.2, 21.3_

  - [x] 2.2 Write property test for VPC module structure
    - Verify module contains required files (main.tf, variables.tf, outputs.tf, versions.tf, README.md)
    - Verify `create` variable exists with type `bool` and default `true`
    - Verify outputs use `try()` function
    - **Property 1: Module File Structure Compliance**
    - **Property 2: Conditional Creation Pattern Compliance**
    - **Validates: Requirements 1.3, 2.1, 2.2, 2.3, 2.4**

- [x] 3. Implement DOKS cluster module
  - [x] 3.1 Create DOKS cluster module structure
    - Create `modules/doks-cluster/main.tf` with `digitalocean_kubernetes_cluster` resource
    - Implement default node pool with autoscaling support
    - Implement `lifecycle { ignore_changes = [version] }` for version drift prevention
    - Implement maintenance window configuration
    - Create `modules/doks-cluster/variables.tf` with all required variables
    - Create `modules/doks-cluster/outputs.tf` with cluster_id, endpoint, token, ca_certificate, kubeconfig
    - Create `modules/doks-cluster/versions.tf` and `README.md`
    - _Requirements: 4.1, 4.2, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11_

  - [x] 3.2 Add additional node pools support
    - Implement `digitalocean_kubernetes_node_pool` resource with `for_each`
    - Support taints, labels, and autoscaling per pool
    - Configure observability node pool for production
    - _Requirements: 4.3, 4.6_

  - [x] 3.3 Write property test for DOKS module
    - Verify module structure compliance
    - Verify conditional creation pattern
    - Verify naming conventions (underscores for identifiers)
    - **Property 1: Module File Structure Compliance**
    - **Property 3: Terraform Identifier Naming Convention**
    - **Validates: Requirements 1.3, 1.8, 2.1**

- [x] 4. Implement managed PostgreSQL module
  - [x] 4.1 Create PostgreSQL module structure
    - Create `modules/managed-postgres/main.tf` with `digitalocean_database_cluster` resource
    - Configure engine `pg`, version 16, VPC association
    - Create separate databases: `platform`, `transaction`
    - Create separate users: `platform_service`, `transaction_service`
    - Create `modules/managed-postgres/variables.tf` with all required variables
    - Create `modules/managed-postgres/outputs.tf` with connection details (marked sensitive)
    - _Requirements: 5.1, 5.2, 5.5, 5.6, 5.8, 5.9_

  - [x] 4.2 Add read replica support
    - Implement `digitalocean_database_replica` resource with conditional creation
    - Output replica connection details separately
    - _Requirements: 5.4, 5.10_

  - [x] 4.3 Add database firewall rules
    - Implement `digitalocean_database_firewall` resource
    - Configure rules to allow only DOKS cluster access (type `k8s`)
    - _Requirements: 5.7, 20.1, 20.3, 20.4_

  - [x] 4.4 Write property test for PostgreSQL module
    - Verify firewall rules only allow k8s type
    - Verify no public access rules exist
    - **Property 6: Database Firewall Isolation**
    - **Validates: Requirements 20.1, 20.2, 20.3, 20.4**

- [x] 5. Implement managed Redis module
  - [x] 5.1 Create Redis module structure
    - Create `modules/managed-redis/main.tf` with `digitalocean_database_cluster` resource
    - Configure engine `redis`, version 7, VPC association
    - Configure eviction policy `allkeys_lru`
    - Create `modules/managed-redis/variables.tf` with all required variables
    - Create `modules/managed-redis/outputs.tf` with connection details (marked sensitive)
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.6_

  - [x] 5.2 Add Redis firewall rules
    - Implement `digitalocean_database_firewall` resource
    - Configure rules to allow only DOKS cluster access
    - _Requirements: 6.5, 20.2_

- [ ] 6. Implement supporting modules
  - [ ] 6.1 Create Spaces module
    - Create `modules/spaces/main.tf` with `digitalocean_spaces_bucket` resource
    - Implement CORS configuration support
    - Implement lifecycle rules support
    - Implement optional CDN with `digitalocean_cdn` resource
    - Create variables.tf, outputs.tf, versions.tf, README.md
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6_

  - [ ] 6.2 Create Container Registry module
    - Create `modules/container-registry/main.tf` with `digitalocean_container_registry` resource
    - Configure `basic` subscription tier, `fra1` region
    - Create variables.tf, outputs.tf, versions.tf, README.md
    - _Requirements: 8.1, 8.2, 8.3_

  - [ ] 6.3 Create DNS module
    - Create `modules/dns/main.tf` with `digitalocean_domain` and `digitalocean_record` resources
    - Support A, CNAME, and TXT records via `for_each`
    - Create variables.tf, outputs.tf, versions.tf, README.md
    - _Requirements: 9.1, 9.2, 9.3, 9.4_

  - [ ] 6.4 Create Project module
    - Create `modules/project/main.tf` with `digitalocean_project` and `digitalocean_project_resources` resources
    - Support associating resource URNs with the project
    - Create variables.tf, outputs.tf, versions.tf, README.md
    - _Requirements: 32.1, 32.2, 32.3_

- [ ] 7. Checkpoint - Verify all modules
  - Ensure all 8 modules (vpc, doks-cluster, managed-postgres, managed-redis, spaces, container-registry, dns, project) pass `terraform validate`
  - Ensure all modules pass TFLint checks
  - Run pre-commit hooks on all files
  - Verify module structure compliance (each module has main.tf, variables.tf, outputs.tf, versions.tf, README.md)
  - Verify each module has `create` variable with `bool` type and `true` default
  - Ask the user if questions arise.

- [ ] 8. Create environment compositions
  - [ ] 8.1 Create shared environment composition
    - Create `environments/shared/main.tf` calling all modules (vpc, doks, postgres, redis, project)
    - Configure VPC with IP range `10.10.0.0/16`
    - Configure DOKS with 2-4 nodes, HA disabled, autoscaling enabled
    - Configure PostgreSQL with no read replica
    - Configure Redis with firewall rules
    - Configure Project module to group all shared resources
    - Configure maintenance windows: DOKS Sunday 04:00, PostgreSQL Sunday 04:00, Redis Sunday 04:00
    - Create `environments/shared/variables.tf` and `outputs.tf`
    - Create `environments/shared/terraform.tfvars` with environment-specific values
    - _Requirements: 3.2, 4.4, 5.3, 11.1, 11.3, 11.4, 29.1, 29.2, 29.3, 29.5, 32.1, 32.2, 35.1, 35.2, 35.3_

  - [ ] 8.2 Create production environment composition
    - Create `environments/production/main.tf` calling all modules (vpc, doks, postgres, redis, dns, project)
    - Configure VPC with IP range `10.20.0.0/16`
    - Configure DOKS with 3-6 nodes, HA enabled, observability node pool
    - Configure PostgreSQL with read replica
    - Configure Redis with firewall rules
    - Configure Project module to group all production resources
    - Configure maintenance windows: DOKS Sunday 03:00, PostgreSQL Sunday 03:00, Redis Sunday 03:00
    - Create `environments/production/variables.tf` and `outputs.tf`
    - Create `environments/production/terraform.tfvars` with environment-specific values
    - _Requirements: 3.3, 4.5, 4.6, 5.4, 11.2, 11.3, 11.4, 29.1, 29.2, 29.3, 29.4, 32.1, 32.2_

  - [ ]* 8.3 Write property test for resource tagging
    - Verify all resources have required tags
    - **Property 5: Resource Tagging Compliance**
    - **Validates: Requirements 21.1, 21.2, 21.3, 21.4**

- [ ] 9. Implement GitHub Actions CI/CD pipelines
  - [ ] 9.1 Create PR workflow
    - Create `.github/workflows/terraform-pr.yml`
    - Implement `terraform plan` for both shared and production environments
    - Post plan output as PR comment (truncated to 60KB)
    - Upload plan artifacts for later use
    - Configure manual approval gate for dev deployment
    - _Requirements: 13.1, 13.3, 13.4, 13.5, 13.9_

  - [ ] 9.2 Create merge/deploy workflow
    - Create `.github/workflows/terraform-deploy.yml`
    - Implement auto-deploy to test after merge
    - Implement manual approval gates for staging and production
    - Re-plan before each deployment to catch drift
    - Use saved plan artifacts for apply
    - Pin Terraform version 1.9.0
    - _Requirements: 13.2, 13.6, 13.7, 13.8, 13.10, 13.11_

- [ ] 10. Create Kubernetes manifests structure
  - [ ] 10.1 Create Kustomize base structure
    - Create `kubernetes/base/namespaces/` with namespace definitions
    - Create `kubernetes/base/nginx-ingress/` with namespace and kustomization
    - Create `kubernetes/base/cert-manager/` with namespace, ClusterIssuer, kustomization
    - Create `kubernetes/base/sealed-secrets/` with kustomization
    - _Requirements: 15.1, 15.2, 15.3, 16.2, 17.2, 19.1_

  - [ ] 10.2 Create Kustomize overlays
    - Create `kubernetes/overlays/dev/kustomization.yaml`
    - Create `kubernetes/overlays/test/kustomization.yaml`
    - Create `kubernetes/overlays/staging/kustomization.yaml` with Istio label
    - Create `kubernetes/overlays/production/kustomization.yaml` with observability
    - _Requirements: 15.4, 15.5, 15.6_

  - [ ] 10.3 Create Helm values files
    - Create `kubernetes/helm-values/nginx-ingress-values.yaml` with DO LB annotations
    - Create `kubernetes/helm-values/cert-manager-values.yaml`
    - Create `kubernetes/helm-values/prometheus-values.yaml` with resource sizing
    - Create `kubernetes/helm-values/grafana-values.yaml`
    - Create `kubernetes/helm-values/jaeger-values.yaml`
    - Create `kubernetes/helm-values/loki-values.yaml`
    - Create `kubernetes/helm-values/sealed-secrets-values.yaml`
    - _Requirements: 16.6, 18.1, 18.2, 18.3, 18.4, 18.5, 18.6_

- [ ] 11. Checkpoint - Verify Kubernetes manifests
  - Run `kubectl apply --dry-run=client` on all manifests
  - Verify Helm values are valid YAML
  - Ask the user if questions arise.

- [ ] 12. Create Docker Compose for local development
  - [ ] 12.1 Create docker-compose.yml
    - Create `docker-compose.yml` at the infrastructure repository root (separate from the workspace-level docker-compose.yml)
    - Add PostgreSQL with PostGIS extension (postgis/postgis:15-3.3)
    - Add Redis (redis:7-alpine)
    - Add Kafka in KRaft mode (confluentinc/cp-kafka:7.6.0)
    - Configure ports: PostgreSQL 5432, Redis 6379, Kafka 9092
    - Configure named volumes for data persistence
    - _Requirements: 22.1, 22.2, 22.3, 22.4, 22.5, 22.6, 22.7, 22.8_

- [ ] 13. Create operational scripts
  - [ ] 13.1 Create backup verification script
    - Create `scripts/backup-verify.sh`
    - Implement backup restoration to temporary instance
    - Implement data integrity validation
    - Document usage instructions
    - _Requirements: 28.5, 28.6, 30.1_

  - [ ] 13.2 Create secret rotation script
    - Create `scripts/secret-rotation.sh`
    - Implement database password rotation
    - Implement API key rotation
    - Document usage instructions
    - _Requirements: 30.2, 30.4_

  - [ ] 13.3 Create cluster maintenance script
    - Create `scripts/cluster-maintenance.sh`
    - Implement DOKS cluster operations using doctl
    - Document usage instructions
    - _Requirements: 30.3, 30.4, 30.5_

- [ ] 14. Create validation scripts
  - [ ] 14.1 Create module structure validation script
    - Create `scripts/validate-module-structure.sh`
    - Verify each module has required files
    - Verify `create` variable exists in each module
    - Verify outputs use `try()` function
    - _Requirements: 1.3, 2.1, 2.3_

  - [ ] 14.2 Create naming convention validation script
    - Create `scripts/validate-naming-conventions.sh`
    - Verify Terraform identifiers use underscores
    - Verify cloud resource names use dashes
    - _Requirements: 1.8, 1.9_

- [ ] 15. Create documentation
  - [ ] 15.1 Create external services setup documentation
    - Document Redpanda Serverless setup (topics, credentials)
    - Document Stripe account setup
    - Document SendGrid setup
    - Document Firebase Cloud Messaging setup
    - Document OpenWeatherMap API setup
    - Document OAuth provider setup (Google, Facebook, Apple)
    - Document which credentials go to GitHub Secrets vs Sealed Secrets
    - _Requirements: 31.1, 31.2, 31.3, 31.4, 31.5, 31.6, 31.7_

  - [ ] 15.2 Create DigitalOcean setup guide
    - Document manual steps in DO console (API token, Spaces bucket, access keys)
    - Document GitHub environments and secrets setup
    - Document container registry creation
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5, 14.6, 14.7_

  - [ ] 15.3 Create repository README
    - Document repository structure
    - Document how to run locally
    - Document CI/CD pipeline flow
    - Document maintenance procedures

- [ ] 16. Final checkpoint - End-to-end validation
  - Run `terraform init` and `terraform validate` on all environments
  - Run TFLint on all modules
  - Run pre-commit hooks
  - Run all validation scripts
  - Verify docker-compose starts successfully
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional property-based tests and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties
- The infrastructure repository should be created first before executing these tasks
- GitHub environments and secrets must be configured manually before CI/CD pipelines can run
- Redpanda Serverless and external services require manual setup in their respective consoles
- Requirements 23-25 and 27 (Spring Boot scaffolding, Flyway migrations, common library, application CI/CD) have been moved to a separate "phase-1-service-scaffolding" spec as they target different repositories
- Database firewall rules are embedded within the managed-postgres and managed-redis modules (not standalone modules)
- Load balancer is created automatically by NGINX Ingress Controller annotations (not a standalone Terraform module)
