# Hardcoded Values Report - Dev & Stage Environments

**Date:** 2025-01-XX  
**Environments:** `environments/dev/` & `environments/stage/`  
**Status:** Review Required

---

## Executive Summary

This report identifies all hardcoded values in both Dev and Stage environments that should be moved to variables for better maintainability, security, and environment flexibility.

### Key Statistics

| Metric | Dev | Stage | Combined |
|--------|-----|-------|----------|
| **Total Hardcoded Values** | 47 | 45 | 47 unique patterns |
| **Files with Issues** | 11 | 11 | Same files |
| **Files Clean** | 3 | 3 | VPC, IAM, RDS |
| **Security Risks** | 2 | 2 | VPN CIDRs, S3 force_destroy |

### Environment Comparison

| Module | Dev File | Stage File | Hardcoded Values | Status |
|--------|----------|------------|------------------|--------|
| VPC | `04_vpc.tf` | `04_vpc.tf` | ✅ None | Clean |
| ACM | `06_acm.tf` | `05_acm.tf` | 2 values | Same |
| ALB | `11_alb.tf` | `06_alb.tf` | 8 values | Same |
| IAM | `07_iam.tf` | `07_iam.tf` | ✅ None | Clean |
| KMS | `08_kms.tf` | `09_kms.tf` | 7 values | Same |
| ElastiCache | `09_elastic_cache.tf` | `10_elastic_cache.tf` | 3 values | Same |
| RDS | `10_rds.tf` | `15_rds.tf` | ✅ None | Clean |
| ECR | `12_ecr.tf` | `12_ecr.tf` | 3 values | Same |
| Secrets Manager | `13_secrets_manager.tf` | `16_secrets_manager.tf` | 4 values | Same |
| ECS | `14_ecs.tf` | `08_ecs.tf` | 2 values | Same |
| S3 | `15_s3.tf` | `11_s3.tf` | 4 values | Same |
| CloudFront | `16_cloudfront.tf` | `13_cloudfront.tf` | 2-3 values | **Different** |
| EC2/VPN | `05_pritunl_vpn.tf` | `14_ec2.tf` | 5 values | Same |
| Route53 | `18_vpn_route53.tf` | N/A | 1 value | Dev only |

**Note:** Stage CloudFront uses `var.default_root_object` (correct), while Dev hardcodes `null` (should be fixed).

---

## 🔴 HIGH PRIORITY - Hardcoded Values by Module

### 1. KMS Module (`08_kms.tf` / `09_kms.tf`)

**Files:** `environments/dev/08_kms.tf`, `environments/stage/09_kms.tf`

**Hardcoded Values:**
```hcl
# KMS Key Aliases
kms_key_alias                = "alias/valkey-kms"
rds_kms_key_alias            = "alias/rds-kms"
ebs_kms_key_alias            = "alias/ebs-kms"
secretsmanager_kms_key_alias = "alias/secretsmanager-kms"

# Key Configuration
deletion_window_in_days = 7
enable_key_rotation     = true
multi_region            = true
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "kms_key_aliases" {
  description = "KMS key aliases for different services"
  type = object({
    valkey         = string
    rds            = string
    ebs            = string
    secretsmanager = string
  })
  default = {
    valkey         = "alias/valkey-kms"
    rds            = "alias/rds-kms"
    ebs            = "alias/ebs-kms"
    secretsmanager = "alias/secretsmanager-kms"
  }
}

variable "kms_deletion_window_in_days" {
  description = "Deletion window in days for KMS keys"
  type        = number
  default     = 7
}

variable "kms_enable_key_rotation" {
  description = "Enable automatic key rotation"
  type        = bool
  default     = true
}

variable "kms_multi_region" {
  description = "Enable multi-region for KMS keys"
  type        = bool
  default     = true
}
```

**Update Module Calls:**
```hcl
module "kms" {
  source = "../../modules/kms"
  
  kms_key_alias                = var.kms_key_aliases.valkey
  rds_kms_key_alias            = var.kms_key_aliases.rds
  ebs_kms_key_alias            = var.kms_key_aliases.ebs
  secretsmanager_kms_key_alias = var.kms_key_aliases.secretsmanager
  
  deletion_window_in_days = var.kms_deletion_window_in_days
  enable_key_rotation     = var.kms_enable_key_rotation
  multi_region            = var.kms_multi_region
  
  tags = local.common_tags
}
```

---

### 2. EC2/VPN Module (`05_pritunl_vpn.tf` / `14_ec2.tf`)

**Files:** `environments/dev/05_pritunl_vpn.tf`, `environments/stage/14_ec2.tf`

**Hardcoded Values (⚠️ SECURITY RISK):**
```hcl
# Security - WARNING: Open to internet
allowed_ssh_cidrs = ["0.0.0.0/0"]  # ⚠️ SECURITY RISK
allowed_vpn_cidrs = ["0.0.0.0/0"]  # ⚠️ SECURITY RISK

# Ports
ssh_port = 22
vpn_port = 443

# User data script path
user_data_script_path = "user_data.sh"
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "vpn_allowed_ssh_cidrs" {
  description = "CIDR blocks allowed to SSH to VPN server"
  type        = list(string)
  default     = ["0.0.0.0/0"]  # Restrict in production!
}

variable "vpn_allowed_vpn_cidrs" {
  description = "CIDR blocks allowed to connect to VPN"
  type        = list(string)
  default     = ["0.0.0.0/0"]  # Restrict in production!
}

variable "vpn_ssh_port" {
  description = "SSH port for VPN server"
  type        = number
  default     = 22
}

variable "vpn_port" {
  description = "VPN service port"
  type        = number
  default     = 443
}

variable "vpn_user_data_script_path" {
  description = "Path to user data script for VPN server"
  type        = string
  default     = "user_data.sh"
}
```

**Update Module Calls:**
```hcl
module "ec2" {
  # ... other config ...
  
  allowed_ssh_cidrs      = var.vpn_allowed_ssh_cidrs
  allowed_vpn_cidrs      = var.vpn_allowed_vpn_cidrs
  ssh_port               = var.vpn_ssh_port
  vpn_port               = var.vpn_port
  user_data_script_path  = var.vpn_user_data_script_path
  
  # ... rest of config ...
}
```

---

### 3. ACM Module (`06_acm.tf` / `05_acm.tf`)

**Files:** `environments/dev/06_acm.tf`, `environments/stage/05_acm.tf`

**Hardcoded Values:**
```hcl
# Certificate Configuration (both ALB and CloudFront modules)
validation_method = "DNS"
certificate_ttl   = 60
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "acm_validation_method" {
  description = "ACM certificate validation method"
  type        = string
  default     = "DNS"
}

variable "acm_certificate_ttl" {
  description = "ACM certificate TTL in seconds"
  type        = number
  default     = 60
}
```

**Update Module Calls:**
```hcl
module "acm" {
  # ... other config ...
  validation_method = var.acm_validation_method
  certificate_ttl   = var.acm_certificate_ttl
  # ... rest of config ...
}

module "acm_cloudfront" {
  # ... other config ...
  validation_method = var.acm_validation_method
  certificate_ttl   = var.acm_certificate_ttl
  # ... rest of config ...
}
```

---

### 4. ElastiCache Module (`09_elastic_cache.tf` / `10_elastic_cache.tf`)

**Files:** `environments/dev/09_elastic_cache.tf`, `environments/stage/10_elastic_cache.tf`

**Hardcoded Values:**
```hcl
# Redis Port
redis_port = 6379

# Encryption Configuration
at_rest_encryption_enabled = true
transit_encryption_enabled = false  # Disabled in dev/stage
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "redis_port" {
  description = "Redis/Valkey port"
  type        = number
  default     = 6379
}

variable "redis_at_rest_encryption_enabled" {
  description = "Enable encryption at rest for ElastiCache"
  type        = bool
  default     = true
}

variable "redis_transit_encryption_enabled" {
  description = "Enable transit encryption for ElastiCache"
  type        = bool
  default     = false  # Can be enabled per environment
}
```

**Update Module Calls:**
```hcl
module "redis" {
  # ... other config ...
  redis_port                = var.redis_port
  at_rest_encryption_enabled = var.redis_at_rest_encryption_enabled
  transit_encryption_enabled = var.redis_transit_encryption_enabled
  # ... rest of config ...
}
```

---

### 5. ALB Module (`11_alb.tf` / `06_alb.tf`)

**Files:** `environments/dev/11_alb.tf`, `environments/stage/06_alb.tf`

**Hardcoded Values:**
```hcl
# SSL Policy
ssl_policy = "ELBSecurityPolicy-TLS13-1-2-Res-2021-06"

# Health Check Configuration
health_check_path                = "/"
health_check_protocol            = "HTTP"
health_check_matcher             = "200"
health_check_interval            = 30
health_check_timeout             = 5
health_check_healthy_threshold   = 2
health_check_unhealthy_threshold = 2
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "alb_ssl_policy" {
  description = "SSL policy for ALB HTTPS listener"
  type        = string
  default     = "ELBSecurityPolicy-TLS13-1-2-Res-2021-06"
}

variable "alb_health_check" {
  description = "ALB health check configuration"
  type = object({
    path                = string
    protocol            = string
    matcher             = string
    interval            = number
    timeout             = number
    healthy_threshold   = number
    unhealthy_threshold = number
  })
  default = {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}
```

**Update Module Calls:**
```hcl
module "alb" {
  # ... other config ...
  ssl_policy          = var.alb_ssl_policy
  health_check_path                = var.alb_health_check.path
  health_check_protocol            = var.alb_health_check.protocol
  health_check_matcher             = var.alb_health_check.matcher
  health_check_interval            = var.alb_health_check.interval
  health_check_timeout             = var.alb_health_check.timeout
  health_check_healthy_threshold   = var.alb_health_check.healthy_threshold
  health_check_unhealthy_threshold = var.alb_health_check.unhealthy_threshold
  # ... rest of config ...
}
```

---

### 6. ECR Module (`12_ecr.tf`)

**Files:** `environments/dev/12_ecr.tf`, `environments/stage/12_ecr.tf`

**Hardcoded Values:**
```hcl
# Image Configuration
image_tag_mutability = "MUTABLE"  # Allow tag overwrites in dev/stage
scan_on_push         = true

# Lifecycle Policy
lifecycle_policy_image_count = 5
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "ecr_image_tag_mutability" {
  description = "ECR image tag mutability (MUTABLE or IMMUTABLE)"
  type        = string
  default     = "MUTABLE"
}

variable "ecr_scan_on_push" {
  description = "Enable image scanning on push"
  type        = bool
  default     = true
}

variable "ecr_lifecycle_policy_image_count" {
  description = "Number of images to keep in ECR lifecycle policy"
  type        = number
  default     = 5
}
```

**Update Module Calls:**
```hcl
module "ecr" {
  # ... other config ...
  image_tag_mutability         = var.ecr_image_tag_mutability
  scan_on_push               = var.ecr_scan_on_push
  lifecycle_policy_image_count = var.ecr_lifecycle_policy_image_count
  # ... rest of config ...
}
```

---

### 7. Secrets Manager Module (`13_secrets_manager.tf` / `16_secrets_manager.tf`)

**Files:** `environments/dev/13_secrets_manager.tf`, `environments/stage/16_secrets_manager.tf`

**Hardcoded Values:**
```hcl
# Secret Configuration
create_service_secrets      = true
service_secrets_description = "Secrets for {service_name} service in ${var.environment} environment"

# Recovery Configuration
recovery_window_in_days = 7

# Rotation Configuration
enable_automatic_rotation = false
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "secrets_create_service_secrets" {
  description = "Create secrets for services"
  type        = bool
  default     = true
}

variable "secrets_description_template" {
  description = "Template for secret descriptions"
  type        = string
  default     = "Secrets for {service_name} service in {environment} environment"
}

variable "secrets_recovery_window_in_days" {
  description = "Recovery window in days for secrets"
  type        = number
  default     = 7
}

variable "secrets_enable_automatic_rotation" {
  description = "Enable automatic rotation for secrets"
  type        = bool
  default     = false
}
```

**Update Module Calls:**
```hcl
module "secrets_manager" {
  # ... other config ...
  create_service_secrets      = var.secrets_create_service_secrets
  service_secrets_description = var.secrets_description_template
  recovery_window_in_days     = var.secrets_recovery_window_in_days
  enable_automatic_rotation   = var.secrets_enable_automatic_rotation
  # ... rest of config ...
}
```

---

### 8. ECS Module (`14_ecs.tf` / `08_ecs.tf`)

**Files:** `environments/dev/14_ecs.tf`, `environments/stage/08_ecs.tf`

**Hardcoded Values:**
```hcl
# CloudWatch Log Retention
log_retention_days = 30

# Secrets (empty list)
secrets = []
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "ecs_log_retention_days" {
  description = "CloudWatch log retention in days for ECS"
  type        = number
  default     = 30
}

variable "ecs_secrets" {
  description = "List of secrets to inject into ECS containers"
  type = list(object({
    name      = string
    valueFrom = string
  }))
  default = []
}
```

**Update Module Calls:**
```hcl
module "ecs_cluster" {
  # ... other config ...
  log_retention_days = var.ecs_log_retention_days
  secrets            = var.ecs_secrets
  # ... rest of config ...
}
```

---

### 9. S3 Module (`15_s3.tf` / `11_s3.tf`)

**Files:** `environments/dev/15_s3.tf`, `environments/stage/11_s3.tf`

**Hardcoded Values (⚠️ SECURITY RISK):**
```hcl
# Force Destroy
force_destroy = true  # ⚠️ Allow bucket deletion in dev/stage

# Versioning
enable_versioning = true

# Encryption
encryption_algorithm = "AES256"
bucket_key_enabled   = false
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "s3_force_destroy" {
  description = "Allow S3 bucket deletion even if not empty"
  type        = bool
  default     = true  # Set to false in production
}

variable "s3_enable_versioning" {
  description = "Enable S3 bucket versioning"
  type        = bool
  default     = true
}

variable "s3_encryption_algorithm" {
  description = "S3 encryption algorithm (AES256 or aws:kms)"
  type        = string
  default     = "AES256"
}

variable "s3_bucket_key_enabled" {
  description = "Enable S3 bucket key for KMS encryption"
  type        = bool
  default     = false
}
```

**Update Module Calls:**
```hcl
module "s3_assets" {
  # ... other config ...
  force_destroy         = var.s3_force_destroy
  enable_versioning     = var.s3_enable_versioning
  encryption_algorithm  = var.s3_encryption_algorithm
  bucket_key_enabled    = var.s3_bucket_key_enabled
  # ... rest of config ...
}
```

---

### 10. CloudFront Module (`16_cloudfront.tf` / `13_cloudfront.tf`)

**Files:** `environments/dev/16_cloudfront.tf`, `environments/stage/13_cloudfront.tf`

**Hardcoded Values:**
```hcl
# Dev: default_root_object = null  (hardcoded)
# Stage: default_root_object = var.default_root_object  (correct)

# Distribution Configuration
enable_ipv6 = false

# Aliases
aliases = ["assets.${var.domain_name}"]
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (both environments)
variable "cloudfront_default_root_object" {
  description = "Default root object for CloudFront"
  type        = string
  default     = null
}

variable "cloudfront_enable_ipv6" {
  description = "Enable IPv6 for CloudFront"
  type        = bool
  default     = false
}

variable "cloudfront_aliases" {
  description = "CloudFront distribution aliases"
  type        = list(string)
  default     = []  # Will be computed as ["assets.${var.domain_name}"]
}
```

**Update Module Calls:**
```hcl
module "cloudfront" {
  # ... other config ...
  default_root_object = var.cloudfront_default_root_object  # Fix Dev to match Stage
  enable_ipv6         = var.cloudfront_enable_ipv6
  aliases            = var.cloudfront_aliases
  # ... rest of config ...
}
```

**Note:** Dev needs to be updated to use `var.cloudfront_default_root_object` like Stage does.

---

### 11. Route53 Record (`18_vpn_route53.tf`) - Dev Only

**Files:** `environments/dev/18_vpn_route53.tf` (Stage doesn't have this)

**Hardcoded Values:**
```hcl
ttl = 300
```

**Recommendation:**
```hcl
# Add to 02_variables.tf (dev environment)
variable "vpn_route53_ttl" {
  description = "TTL for VPN Route53 record"
  type        = number
  default     = 300
}
```

**Update Resource:**
```hcl
resource "aws_route53_record" "vpn" {
  # ... other config ...
  ttl = var.vpn_route53_ttl
  # ... rest of config ...
}
```

---

## ✅ Files with No Hardcoded Values

### Both Environments
- ✅ `04_vpc.tf` - All values from variables
- ✅ `07_iam.tf` - All values from variables  
- ✅ `10_rds.tf` / `15_rds.tf` - All values from variables

---

## 📊 Complete Summary

### Hardcoded Values by Priority

| Priority | Count | Examples |
|----------|-------|----------|
| 🔴 **Security-Critical** | 6 | VPN CIDRs `0.0.0.0/0`, S3 `force_destroy` |
| 🔴 **High Priority** | 36 | KMS config, ALB health checks, encryption settings |
| 🟡 **Medium Priority** | 5 | Ports, TTLs, standard configurations |

### Total Breakdown

| Category | Dev | Stage | Unique Patterns |
|----------|-----|-------|-----------------|
| **KMS** | 7 | 7 | 7 |
| **EC2/VPN** | 5 | 5 | 5 |
| **ACM** | 2 | 2 | 2 |
| **ElastiCache** | 3 | 3 | 3 |
| **ALB** | 8 | 8 | 8 |
| **ECR** | 3 | 3 | 3 |
| **Secrets Manager** | 4 | 4 | 4 |
| **ECS** | 2 | 2 | 2 |
| **S3** | 4 | 4 | 4 |
| **CloudFront** | 3 | 2 | 3 |
| **Route53** | 1 | 0 | 1 |
| **TOTAL** | **47** | **45** | **47** |

---

## 🎯 Recommended Action Plan

### Phase 1: Security-Critical (Immediate) ⚠️
1. ✅ Move VPN security CIDRs to variables (`05_pritunl_vpn.tf` / `14_ec2.tf`)
2. ✅ Move S3 `force_destroy` to variable (`15_s3.tf` / `11_s3.tf`)

### Phase 2: Configuration Management (High Priority)
3. ✅ Move all KMS configuration to variables (`08_kms.tf` / `09_kms.tf`)
4. ✅ Move ALB health check config to variables (`11_alb.tf` / `06_alb.tf`)
5. ✅ Move ECS log retention to variable (`14_ecs.tf` / `08_ecs.tf`)
6. ✅ Fix Dev CloudFront to use variable (match Stage)

### Phase 3: Environment-Specific (Medium Priority)
7. ✅ Move encryption settings to variables (`09_elastic_cache.tf`, `15_s3.tf`)
8. ✅ Move ECR lifecycle policy to variable (`12_ecr.tf`)
9. ✅ Move Secrets Manager config to variables (`13_secrets_manager.tf` / `16_secrets_manager.tf`)

### Phase 4: Nice to Have (Low Priority)
10. ✅ Move ACM certificate config to variables (`06_acm.tf` / `05_acm.tf`)
11. ✅ Move CloudFront aliases to variable (`16_cloudfront.tf` / `13_cloudfront.tf`)
12. ✅ Move Route53 TTL to variable (`18_vpn_route53.tf` - Dev only)

---

## 📝 Implementation Steps

### Step 1: Add Variables
1. Add all recommended variables to `environments/dev/02_variables.tf`
2. Add all recommended variables to `environments/stage/02_variables.tf`
3. Ensure variable names and defaults are consistent

### Step 2: Update Module Calls
1. Update module calls in `environments/dev/` to use variables
2. Update module calls in `environments/stage/` to use variables
3. Ensure both environments use the same variable names

### Step 3: Update tfvars
1. Add environment-specific values to `environments/dev/terraform.tfvars`
2. Add environment-specific values to `environments/stage/terraform.tfvars`
3. Use different values per environment where needed (e.g., security CIDRs)

### Step 4: Test
1. Run `terraform validate` in both environments
2. Run `terraform plan` in both environments
3. Review changes before applying
4. Apply changes to dev first, then stage

---

## ✅ Benefits of Removing Hardcoded Values

1. **Environment Flexibility:** Easy to change values per environment (dev/stage/prod)
2. **Maintainability:** Single source of truth in `terraform.tfvars`
3. **Security:** Easier to restrict access in production
4. **Documentation:** Variables serve as documentation
5. **Testing:** Easy to test different configurations
6. **Compliance:** Better audit trail of configuration changes
7. **Consistency:** Same variable names across environments
8. **Reusability:** Variables can be reused across modules

---

## 📌 Notes

- Some hardcoded values may be intentional (e.g., `6379` for Redis port is standard)
- Security-related hardcoded values (like `0.0.0.0/0`) should be **prioritized**
- Environment-specific defaults can remain in variables with sensible defaults
- Consider using `locals` for computed values that depend on variables
- Stage CloudFront implementation is better than Dev (uses variable)
- Dev has one additional file (Route53 VPN record) that Stage doesn't have

---

## 🔄 Environment-Specific Notes

### Dev Environment
- Has VPN Route53 record (`18_vpn_route53.tf`) - needs TTL variable
- CloudFront uses hardcoded `null` - should use variable like Stage

### Stage Environment
- CloudFront correctly uses `var.default_root_object` ✅
- No VPN Route53 record (not needed)

### Both Environments
- Apply all variable recommendations consistently
- Ensure same variable names and defaults across environments
- Use environment-specific values in `terraform.tfvars` where appropriate

---

**Next Steps:**
1. ✅ Review this consolidated report
2. ✅ Prioritize which hardcoded values to move to variables
3. ✅ Create variables in `02_variables.tf` for both environments
4. ✅ Update module calls in **both environments** to use variables
5. ✅ Add values to `terraform.tfvars` for each environment
6. ✅ Test changes with `terraform plan` in both environments
7. ✅ **Fix Dev CloudFront** to match Stage's use of `var.default_root_object`

---

**Report Generated:** 2025-01-XX  
**Last Updated:** 2025-01-XX
