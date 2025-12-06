#  AWS to Azure Migration Plan

## Executive Summary

This document outlines a comprehensive migration strategy for replicating the existing AWS infrastructure to Microsoft Azure with minimal downtime and optimized time-to-completion. The migration encompasses compute resources, networking, database, and application assets including static files (images, PDFs, etc.).

**Estimated Timeline**: 2-3 weeks  
**Estimated Downtime**: Less than 2 hours (during final cutover)  
**Migration Strategy**: Parallel run with gradual cutover

## Current AWS Architecture Analysis

### Existing Components
- **Compute**: 2x EC2 t2.micro instances (Frontend: Uptime Kuma, Backend: Laravel)
- **Database**: RDS MySQL 8.0 (Multi-AZ, 20GB)
- **Networking**: VPC with 2 public and 2 private subnets across 2 AZs
- **Security**: Security Groups, Network ACLs
- **Monitoring**: CloudWatch with SNS alerts
- **CI/CD**: GitHub Actions workflows

### Azure Equivalent Mapping

| AWS Service | Azure Equivalent | Notes |
|-------------|------------------|-------|
| EC2 | Azure Virtual Machines (B1s/B2s) | Similar CPU/RAM specs |
| RDS MySQL | Azure Database for MySQL | Managed service, similar features |
| VPC | Virtual Network (VNet) | Native Azure networking |
| Subnets | Subnets | Direct 1:1 mapping |
| Security Groups | Network Security Groups (NSG) | Stateful firewall rules |
| Network ACLs | NSG (subnet-level) | Can implement both instance and subnet rules |
| Internet Gateway | Virtual Network Gateway | Managed by Azure |
| CloudWatch | Azure Monitor | Metrics, logs, and alerts |
| SNS | Azure Monitor Action Groups | Alert notifications |
| EBS Volumes | Azure Managed Disks | Standard SSD recommended |

## Migration Plan

### Phase 1: Pre-Migration Preparation (Week 1)

#### 1.1 Azure Account Setup
**Duration**: 1 day

- Create Azure subscription
- Set up Azure Active Directory
- Configure billing and cost management
- Establish resource groups:
  - `rg-obelion-network`
  - `rg-obelion-compute`
  - `rg-obelion-database`
  - `rg-obelion-monitoring`

#### 1.2 Infrastructure as Code Migration
**Duration**: 2-3 days

**Approach**: Convert Terraform AWS modules to Azure

```
terraform-azure/
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    ├── 01-vnet/              # Azure Virtual Network
    ├── 02-subnets/           # Subnets configuration
    ├── 03-nsg/               # Network Security Groups
    ├── 04-virtual-machines/  # VM instances
    ├── 05-mysql/             # Azure Database for MySQL
    └── 06-monitoring/        # Azure Monitor setup
```

**Key Terraform Changes**:
- Provider: `azurerm` instead of `aws`
- Resource naming: `azurerm_virtual_network` vs `aws_vpc`
- Authentication: Azure Service Principal or Managed Identity
- State management: Azure Storage Account backend

#### 1.3 Network Architecture Design
**Duration**: 1 day

**Azure Network Configuration**:
```
Virtual Network: 10.0.0.0/16 (same CIDR to minimize DNS/routing changes)
├── Public Subnet 1: 10.0.1.0/24 (West Europe Zone 1)
├── Public Subnet 2: 10.0.2.0/24 (West Europe Zone 2)
├── Private Subnet 1: 10.0.3.0/24 (West Europe Zone 1)
└── Private Subnet 2: 10.0.4.0/24 (West Europe Zone 2)
```

**Network Security Groups**:
- Frontend NSG: Allow 80, 443, 22 inbound
- Backend NSG: Allow 80 from Frontend NSG, 22 from management
- Database NSG: Allow 3306 from Backend NSG only

### Phase 2: Infrastructure Deployment (Week 1-2)

#### 2.1 Deploy Azure Infrastructure
**Duration**: 1-2 days

**Steps**:
1. Deploy Virtual Network and subnets
2. Create Network Security Groups
3. Deploy Virtual Machines:
   - Frontend VM: Standard B1s (1 vCPU, 1GB RAM)
   - Backend VM: Standard B1s (1 vCPU, 1GB RAM)
4. Attach Managed Disks (8GB Standard SSD)
5. Configure Azure Monitor alerts

**Terraform Execution**:
```bash
cd terraform-azure
terraform init -backend-config="storage_account_name=obelionterraform"
terraform plan
terraform apply
```

#### 2.2 Database Migration Strategy
**Duration**: 2-3 days

**Option A: Online Migration (Recommended - Zero Downtime)**

Using **Azure Database Migration Service (DMS)**:

1. **Setup Phase**:
   - Create Azure Database for MySQL (Flexible Server)
   - Configure server parameters to match RDS settings
   - Enable SSL/TLS connections
   - Configure firewall rules

2. **Migration Phase**:
   ```bash
   # Create DMS instance
   az dms project create \
     --resource-group rg-obelion-database \
     --service-name obelion-dms \
     --source-platform MySQL \
     --target-platform AzureDbForMySQL
   
   # Configure continuous sync
   az dms task create \
     --task-type OnlineMigration \
     --source-connection-json source.json \
     --target-connection-json target.json
   ```

3. **Validation Phase**:
   - Monitor replication lag (aim for <1 second)
   - Validate data integrity with checksums
   - Test read operations on Azure database

4. **Cutover Phase**:
   - Stop application writes to AWS RDS
   - Wait for replication to complete
   - Update application configuration
   - Resume operations on Azure

**Option B: Offline Migration (Backup/Restore)**

If downtime is acceptable:

```bash
# On AWS RDS
mysqldump -h rds-endpoint.amazonaws.com \
  -u admin -p \
  --single-transaction \
  --routines \
  --triggers \
  --databases mydb > backup.sql

# Compress for faster transfer
gzip backup.sql

# Transfer to Azure VM
az storage blob upload \
  --account-name obeliondatastorage \
  --container-name migrations \
  --file backup.sql.gz

# On Azure Database
mysql -h obelion-mysql.mysql.database.azure.com \
  -u admin -p < backup.sql
```

**Expected Downtime**: 30-60 minutes for 20GB database

#### 2.3 Application Deployment
**Duration**: 2 days

**Frontend (Uptime Kuma)**:
1. Install Docker on Azure VM:
   ```bash
   # user_data script
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   sudo usermod -aG docker azureuser
   ```

2. Deploy application:
   ```bash
   cd ~/app
   git clone https://github.com/Omarh4700/uptime-kuma.git .
   docker-compose up -d
   ```

**Backend (Laravel)**:
1. Install PHP 8.3 and dependencies
2. Configure Apache/Nginx
3. Update `.env` with Azure DB credentials:
   ```env
   DB_HOST=obelion-mysql.mysql.database.azure.com
   DB_PORT=3306
   DB_DATABASE=mydb
   ```

### Phase 3: Asset Migration (Week 2)

#### 3.1 Static Asset Transfer Strategy

**Current State**: Application assets stored on EC2 instance filesystem

**Target State**: Azure Blob Storage for scalability

**Migration Steps**:

1. **Create Azure Storage Account**:
   ```bash
   az storage account create \
     --name obelionstorage \
     --resource-group rg-obelion-compute \
     --location westeurope \
     --sku Standard_LRS \
     --kind StorageV2
   ```

2. **Create Blob Containers**:
   ```bash
   az storage container create \
     --name product-images \
     --account-name obelionstorage
   
   az storage container create \
     --name documents \
     --account-name obelionstorage
   ```

3. **Transfer Assets**:
   
   **Method 1: AzCopy (Recommended for large transfers)**
   ```bash
   # On AWS EC2
   wget https://aka.ms/downloadazcopy-v10-linux
   tar -xvf downloadazcopy-v10-linux
   
   # Sync assets
   ./azcopy sync '/path/to/assets' \
     'https://obelionstorage.blob.core.windows.net/product-images?[SAS-Token]' \
     --recursive
   ```

   **Method 2: Azure CLI**
   ```bash
   az storage blob upload-batch \
     --destination product-images \
     --source /path/to/local/assets \
     --account-name obelionstorage
   ```

4. **Update Application Code**:
   - Modify file upload logic to use Azure Storage SDK
   - Update file retrieval URLs to Blob Storage endpoints
   - Configure CDN if needed (Azure CDN)

**Expected Transfer Time**:
- 10GB of images: ~30-60 minutes (depending on bandwidth)
- Parallel uploads can reduce time by 50%

#### 3.2 Application Configuration Updates

Update Laravel storage configuration:
```php
// config/filesystems.php
'azure' => [
    'driver' => 'azure',
    'container' => 'product-images',
    'account-name' => env('AZURE_STORAGE_ACCOUNT'),
    'account-key' => env('AZURE_STORAGE_KEY'),
]
```

### Phase 4: CI/CD Migration (Week 2)

#### 4.1 GitHub Actions Workflow Updates

**Frontend Workflow** (`azure-deploy-frontend.yml`):
```yaml
name: Deploy Frontend to Azure

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Deploy to VM
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.AZURE_VM_HOST }}
          username: azureuser
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/app
            git pull origin master
            docker-compose up -d
```

**Backend Workflow** (`azure-deploy-backend.yml`):
```yaml
name: Deploy Backend to Azure

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Deploy Application
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.AZURE_VM_HOST }}
          username: azureuser
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/app
            git pull origin main
            composer install --no-dev
            php artisan migrate --force
            sudo systemctl restart apache2
```

**Required GitHub Secrets**:
- `AZURE_CREDENTIALS`: Service principal JSON
- `AZURE_VM_HOST`: VM public IP
- `SSH_PRIVATE_KEY`: SSH key for VM access
- `AZURE_DB_HOST`: Azure MySQL hostname
- `AZURE_DB_PASSWORD`: Database password

### Phase 5: Testing & Validation (Week 2-3)

#### 5.1 End-to-End Testing
**Duration**: 2-3 days

**Test Checklist**:
- [ ] Frontend application loads successfully
- [ ] User authentication works
- [ ] Backend API endpoints respond correctly
- [ ] Database read/write operations function
- [ ] Static assets (images, PDFs) load from Blob Storage
- [ ] Monitoring alerts trigger correctly
- [ ] CI/CD deployments succeed
- [ ] SSL/TLS certificates valid (if using HTTPS)

#### 5.2 Performance Testing

**Load Testing**:
```bash
# Using Apache Bench
ab -n 1000 -c 10 http://azure-frontend-ip/

# Using Artillery
artillery quick --count 100 --num 10 http://azure-frontend-ip/
```

**Database Performance**:
```sql
-- Query execution time comparison
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

#### 5.3 Disaster Recovery Validation

Test backup and restore procedures:
```bash
# MySQL automated backups (Azure handles this)
az mysql flexible-server backup list \
  --resource-group rg-obelion-database \
  --server-name obelion-mysql

# Point-in-time restore test
az mysql flexible-server restore \
  --source-server obelion-mysql \
  --name obelion-mysql-restore-test \
  --restore-time "2024-12-05T20:00:00Z"
```

### Phase 6: Cutover & Go-Live (Week 3)

#### 6.1 Pre-Cutover Checklist

**48 Hours Before**:
- [ ] Final full backup of AWS RDS
- [ ] Verify Azure infrastructure is 100% functional
- [ ] Update DNS TTL to 300 seconds (5 minutes)
- [ ] Notify stakeholders of maintenance window
- [ ] Create rollback runbook

**24 Hours Before**:
- [ ] Sync database one final time
- [ ] Verify monitoring dashboards
- [ ] Prepare communication templates
- [ ] Assign on-call team

#### 6.2 Cutover Procedure

**Timeline**: 2-hour maintenance window (recommended Saturday 2 AM UTC)

**Minute 0-10**: Application Freeze
```bash
# On AWS
# Set maintenance mode
php artisan down --message="Migrating to Azure"

# Stop write operations
sudo systemctl stop apache2
```

**Minute 10-30**: Final Database Sync
```bash
# Verify replication lag
SELECT * FROM azure_dms_replication_status;

# Ensure lag < 1 second
# Stop replication
az dms task cutover
```

**Minute 30-60**: DNS Update
```bash
# Update DNS records
# A record: frontend.obelion.com -> Azure VM IP
# CNAME: api.obelion.com -> Azure VM IP

# Verify propagation
dig frontend.obelion.com
```

**Minute 60-90**: Application Startup on Azure
```bash
# On Azure VM
sudo systemctl start apache2
php artisan up

# Smoke tests
curl -I http://azure-frontend-ip/
curl http://azure-backend-ip/api/health
```

**Minute 90-120**: Monitoring & Validation
- Monitor error rates in Azure Monitor
- Check database connections
- Verify user access
- Test critical user flows

#### 6.3 Rollback Plan

If issues occur during cutover:

**Immediate Rollback** (<30 minutes):
```bash
# Revert DNS to AWS IPs
# Restart AWS services
sudo systemctl start apache2
php artisan up

# Communicate rollback to users
```

**Post-Cutover Rollback** (>30 minutes):
- Continue on Azure but with known issues documented
- Fix forward rather than rollback
- Use Azure's built-in redundancy

### Phase 7: Post-Migration (Week 3+)

#### 7.1 Optimization

**Cost Optimization**:
- Review Azure Advisor recommendations
- Right-size VMs based on actual usage
- Enable Azure Reserved Instances (1-year commitment for 40% savings)
- Implement auto-shutdown for non-production resources

**Performance Optimization**:
- Enable Azure CDN for static assets
- Configure Application Gateway for load balancing
- Implement Redis Cache (Azure Cache for Redis)
- Enable query performance insights on MySQL

#### 7.2 Decommissioning AWS Resources

**Week 4** (after 7 days stable operation):
1. Create final AWS snapshots
2. Stop EC2 instances (don't terminate yet)
3. Stop RDS instance
4. Retain snapshots for 30 days

**Week 8** (after 30 days):
1. Terminate EC2 instances
2. Delete RDS instance (keep final snapshot)
3. Remove VPC and networking
4. Cancel AWS subscription

## Downtime Minimization Strategy

### Achieving <2 Hours Downtime

**Key Techniques**:

1. **Database Continuous Replication**: Use Azure DMS online migration to keep databases in sync
2. **Parallel Infrastructure**: Build Azure environment while AWS runs
3. **DNS-Based Cutover**: Change DNS records instead of moving data during maintenance
4. **Pre-warming**: Run tests on Azure to ensure all services are ready
5. **Staged Rollout**: Optionally use weighted DNS routing (10% to Azure, 90% to AWS, then gradually shift)

### Alternative: Blue-Green Deployment

If zero downtime is critical:

```
Phase 1: Azure = Blue (new), AWS = Green (current, 100% traffic)
Phase 2: Testing on Blue environment
Phase 3: Route 10% traffic to Blue
Phase 4: Monitor errors, if acceptable, route 50% to Blue
Phase 5: Route 100% to Blue
Phase 6: Decommission Green
```

## Risk Mitigation

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Data loss during migration | Low | Critical | Multiple backups, checksums, validation |
| Extended downtime | Medium | High | Comprehensive testing, rollback plan |
| Performance degradation | Medium | Medium | Load testing, proper VM sizing |
| Application incompatibility | Low | High | Full testing in Azure before cutover |
| DNS propagation delay | Medium | Low | Lower TTL in advance, CloudFlare proxy |

### Business Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| User disruption | Medium | High | Off-peak migration window, communication |
| Budget overrun | Low | Medium | Cost estimation, monitoring during migration |
| Vendor lock-in | Medium | Low | Use Terraform for portability |

## Success Criteria

**Migration Complete When**:
-  All infrastructure running on Azure
-  Database fully migrated with zero data loss
-  All static assets accessible from Azure Blob Storage
-  CI/CD pipelines deploying to Azure successfully
-  Monitoring and alerts functional
-  Performance metrics equal or better than AWS
-  No critical bugs post-migration
-  AWS resources decommissioned

**KPIs to Monitor**:
- Application uptime: >99.9%
- Page load time: <2 seconds (same as AWS)
- Database query time: <100ms (same as AWS)
- Failed deployment rate: <5%

## Cost Comparison

### AWS Current Monthly Cost (Estimated)
- 2x EC2 t2.micro: $16
- RDS MySQL Multi-AZ: $30
- Data transfer: $5
- **Total: ~$51/month**

### Azure Projected Monthly Cost
- 2x B1s VMs: $15
- Azure Database for MySQL: $35
- Blob Storage (100GB): $2
- Data transfer: $5
- **Total: ~$57/month**

**Note**: Costs similar, but Azure offers:
- Better integration with Microsoft tools
- Hybrid cloud capabilities
- Enterprise agreements for larger scale

## Conclusion

This migration plan provides a structured approach to moving from AWS to Azure with minimal risk and downtime. By leveraging Azure's native migration tools, maintaining parallel infrastructure, and executing a well-tested cutover plan, we can achieve migration completion within 2-3 weeks with less than 2 hours of total downtime.

The key to success is thorough preparation, comprehensive testing, and having a solid rollback plan. With proper execution, the Azure infrastructure will provide equivalent or better performance while maintaining cost efficiency.
