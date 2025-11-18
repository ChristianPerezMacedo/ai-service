# Disaster Recovery & High Availability

## Overview

### Cos'è e Perché è Importante

**Disaster Recovery (DR)** e **High Availability (HA)** sono due discipline complementari ma distinte nella progettazione di sistemi resilienti:

- **High Availability**: Capacità del sistema di rimanere operativo e accessibile durante guasti parziali, con downtime minimo o nullo
- **Disaster Recovery**: Piano strutturato per ripristinare operazioni dopo un evento catastrofico (region failure, data corruption, attacco ransomware)

Nel contesto del nostro **AI Technical Support System**, DR/HA sono critici perché:

1. **Business Continuity**: Il supporto tecnico deve essere disponibile 24/7 per clienti globali
2. **SLA Commitment**: Garantiamo 99.9% uptime ai clienti enterprise
3. **Data Integrity**: I ticket e le risposte AI devono essere preservati anche in caso di disaster
4. **Compliance**: GDPR richiede business continuity plans documentati

### Quando Applicare DR vs HA

**High Availability** per:
- Guasti hardware temporanei (EC2 instance failure, disk failure)
- Manutenzione programmata (patching, upgrades)
- Picchi di traffico improvvisi
- AZ failures (raro ma possibile)

**Disaster Recovery** per:
- Region-wide outages (estremamente raro)
- Data corruption o deletion accidentale
- Attacchi ransomware o cyber attacks
- Errori di deployment catastrofici

### Architettura High-Level

Nel nostro sistema AWS serverless:

```
Primary Region (eu-south-1 Milano)          Standby Region (eu-west-1 Irlanda)
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│ AZ-1    AZ-2    AZ-3            │         │ AZ-1    AZ-2                    │
│ ┌───┐   ┌───┐   ┌───┐           │         │ ┌───┐   ┌───┐                   │
│ │API│   │API│   │API│  Multi-AZ │         │ │API│   │API│  Standby          │
│ └─┬─┘   └─┬─┘   └─┬─┘           │         │ └───┘   └───┘  (minimal)        │
│   └───────┴───────┘             │         │                                 │
│         ┌───┐                   │         │         ┌───┐                   │
│    ┌────┤RDS├────┐ Multi-AZ     │  Async  │    ┌────┤RDS├ Read Replica      │
│    │    └───┘    │ Failover     │  Repl   │    │    └───┘                   │
│  ┌─┴─┐         ┌─┴─┐            │  ──────>│  ┌─┴─┐                          │
│  │Pri│         │Std│            │         │  │Rpl│                          │
│  └───┘         └───┘            │         │  └───┘                          │
│                                 │         │                                 │
│  DynamoDB Global Tables         │  <────> │  DynamoDB Global Tables         │
│  (Active-Active Replication)    │  Bi-dir │  (Active-Active)                │
│                                 │         │                                 │
│  S3 Buckets                     │  ────>  │  S3 Buckets (CRR)               │
│  (Cross-Region Replication)     │  Async  │                                 │
└─────────────────────────────────┘         └─────────────────────────────────┘
                    │                                     ▲
                    └─────── Route 53 Failover ──────────┘
                         (Health Check Based)
```

## Concetti Fondamentali

### RTO e RPO: Le Metriche Chiave

**Recovery Time Objective (RTO)**: Tempo massimo accettabile di downtime
**Recovery Point Objective (RPO)**: Quantità massima di dati che si può perdere (misurata in tempo)

```
┌───────────────────────────────────────────────────────────┐
│                         Timeline                          │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  Last Backup          Disaster         Recovery          │
│      ↓                   ↓             Complete          │
│      │                   │                ↓              │
│  ────●───────────────────✗────────────────●────>         │
│      │◄─── RPO ─────────►│                               │
│      │                   │◄──── RTO ─────►│              │
│                                                           │
│  RPO = Data Loss Window                                   │
│  RTO = Downtime Window                                    │
└───────────────────────────────────────────────────────────┘
```

**Target per il Nostro Sistema**:

| Servizio | RTO | RPO | Rationale |
|----------|-----|-----|-----------|
| API Gateway | 0 min | 0 | Managed service, multi-AZ automatico |
| Lambda Functions | 0 min | 0 | Stateless, multi-AZ automatico |
| DynamoDB | < 1 min | < 1 sec | Global Tables con replica bi-direzionale |
| S3 Knowledge Base | 0 min | 0 | 99.999999999% durability |
| RDS Analytics | 5 min | 5 min | Multi-AZ failover automatico |
| OpenSearch | 30 min | 1 hour | Restore da snapshot giornaliero |
| SageMaker Endpoint | 1 hour | N/A | Re-deploy da Model Registry |

### Strategie di Disaster Recovery

AWS definisce 4 strategie DR, ordinate per RTO crescente:

#### 1. Backup and Restore (RTO: Hours-Days, RPO: Hours)

**Costo**: Minimo
**Complessità**: Bassa

```
Primary Region                  DR Region (cold)
┌─────────────┐                ┌─────────────┐
│   Active    │   Backups      │   Empty     │
│   System    │ ──────────────>│   Region    │
│             │   Scheduled    │             │
└─────────────┘                └─────────────┘
                                      │
                               In case of disaster:
                                      ↓
                               1. Create infrastructure
                               2. Restore from backup
                               3. Point DNS
```

**Quando usarlo**: Sistemi non-critici, reporting databases, log archives

**Nel nostro sistema**: OpenSearch knowledge base (daily snapshots → S3 → restore se necessario)

#### 2. Pilot Light (RTO: Minutes-Hours, RPO: Minutes)

**Costo**: Basso-Medio
**Complessità**: Media

```
Primary Region                  DR Region (pilot light)
┌─────────────┐                ┌─────────────┐
│   Active    │   Continuous   │   Minimal   │
│   System    │   Data Repl    │   Core Only │
│  ┌─────┐    │ ──────────────>│   ┌─────┐   │
│  │ DB  │    │                │   │ DB  │   │
│  └─────┘    │                │   └─────┘   │
└─────────────┘                └─────────────┘
                                      │
                               In case of disaster:
                                      ↓
                               1. Scale up instances
                               2. Deploy app tier
                               3. Point DNS
```

**Quando usarlo**: Business-critical systems con RTO < 1 hour

**Nel nostro sistema**: RDS read replica in eu-west-1 (sempre aggiornato, può essere promosso)

#### 3. Warm Standby (RTO: Minutes, RPO: Seconds)

**Costo**: Medio-Alto
**Complessità**: Alta

```
Primary Region                  DR Region (warm standby)
┌─────────────┐                ┌─────────────┐
│   Active    │   Continuous   │   Running   │
│   System    │   Data Repl    │   Minimal   │
│  (100%)     │ ──────────────>│   Scale     │
│             │                │   (20-30%)  │
└─────────────┘                └─────────────┘
                                      │
                               In case of disaster:
                                      ↓
                               1. Scale to 100%
                               2. Point DNS
                               3. Ready in minutes
```

**Quando usarlo**: Mission-critical systems, RTO < 15 minutes

**Nel nostro sistema**: Non implementato al MVP, pianificato per Phase 2

#### 4. Active-Active (RTO: None, RPO: None)

**Costo**: Massimo
**Complessità**: Molto Alta

```
Primary Region                  Secondary Region
┌─────────────┐                ┌─────────────┐
│   Active    │   Bi-dir       │   Active    │
│   System    │   Replication  │   System    │
│  (50% load) │ ◄────────────► │  (50% load) │
│             │                │             │
└─────────────┘                └─────────────┘
       │                              │
       └──────── Route 53 ────────────┘
              (Geoproximity/Latency)
```

**Quando usarlo**: Zero-downtime requirements, global applications

**Nel nostro sistema**: DynamoDB Global Tables (active-active su eu-south-1 e eu-west-1)

### Terminologia Chiave

**Multi-AZ**: Deployment su multiple Availability Zones nella stessa region
**Cross-Region**: Deployment su multiple AWS regions geograficamente distanti
**Failover**: Switch automatico da risorsa primaria a standby
**Failback**: Ritorno alla risorsa primaria dopo risoluzione del problema
**Health Check**: Monitoraggio continuo dello stato delle risorse
**Blast Radius**: Scope dell'impatto di un singolo punto di failure
**Split-Brain**: Scenario in cui due sistemi si credono entrambi primari (da evitare!)

## Implementazione Pratica

### Esempio 1: Multi-AZ Setup - Architettura Completa

Implementiamo l'architettura multi-AZ per il nostro AI Technical Support System.

#### CloudFormation Template - VPC Multi-AZ

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Multi-AZ VPC for AI Technical Support System'

Parameters:
  EnvironmentName:
    Type: String
    Default: production
    AllowedValues: [development, staging, production]

Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'
        - Key: Environment
          Value: !Ref EnvironmentName

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-igw'

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  # Public Subnets (3 AZs)
  PublicSubnetAZ1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-public-az1'
        - Key: Type
          Value: Public

  PublicSubnetAZ2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-public-az2'

  PublicSubnetAZ3:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: !Select [2, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-public-az3'

  # Private Subnets (3 AZs) - for Lambda functions
  PrivateSubnetAZ1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.11.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-private-az1'
        - Key: Type
          Value: Private

  PrivateSubnetAZ2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.12.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-private-az2'

  PrivateSubnetAZ3:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.13.0/24
      AvailabilityZone: !Select [2, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-private-az3'

  # Data Subnets (3 AZs) - for RDS, OpenSearch
  DataSubnetAZ1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.21.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-data-az1'
        - Key: Type
          Value: Data

  DataSubnetAZ2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.22.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-data-az2'

  DataSubnetAZ3:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.23.0/24
      AvailabilityZone: !Select [2, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-data-az3'

  # NAT Gateways (one per AZ for HA)
  NatGatewayEIPAZ1:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-nat-eip-az1'

  NatGatewayAZ1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIPAZ1.AllocationId
      SubnetId: !Ref PublicSubnetAZ1
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-nat-az1'

  NatGatewayEIPAZ2:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway
    Properties:
      Domain: vpc

  NatGatewayAZ2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIPAZ2.AllocationId
      SubnetId: !Ref PublicSubnetAZ2
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-nat-az2'

  # Route Tables
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-public-rt'

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  # Associate public subnets with public route table
  PublicSubnetRouteTableAssociationAZ1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetAZ1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetRouteTableAssociationAZ2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetAZ2
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetRouteTableAssociationAZ3:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetAZ3
      RouteTableId: !Ref PublicRouteTable

  # Private Route Tables (one per AZ, each with its own NAT Gateway)
  PrivateRouteTableAZ1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-private-rt-az1'

  PrivateRouteAZ1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableAZ1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayAZ1

  PrivateSubnetRouteTableAssociationAZ1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnetAZ1
      RouteTableId: !Ref PrivateRouteTableAZ1

  PrivateRouteTableAZ2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-private-rt-az2'

  PrivateRouteAZ2:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableAZ2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayAZ2

  PrivateSubnetRouteTableAssociationAZ2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnetAZ2
      RouteTableId: !Ref PrivateRouteTableAZ2

  # Data subnet route tables (no internet access)
  DataRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-data-rt'

  DataSubnetRouteTableAssociationAZ1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref DataSubnetAZ1
      RouteTableId: !Ref DataRouteTable

  DataSubnetRouteTableAssociationAZ2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref DataSubnetAZ2
      RouteTableId: !Ref DataRouteTable

  DataSubnetRouteTableAssociationAZ3:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref DataSubnetAZ3
      RouteTableId: !Ref DataRouteTable

  # VPC Endpoints for AWS Services (no NAT Gateway cost)
  S3VPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.s3'
      RouteTableIds:
        - !Ref PrivateRouteTableAZ1
        - !Ref PrivateRouteTableAZ2
        - !Ref DataRouteTable
      VpcEndpointType: Gateway

  DynamoDBVPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.dynamodb'
      RouteTableIds:
        - !Ref PrivateRouteTableAZ1
        - !Ref PrivateRouteTableAZ2
      VpcEndpointType: Gateway

Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${EnvironmentName}-VPC-ID'

  PrivateSubnets:
    Description: List of private subnet IDs
    Value: !Join [',', [!Ref PrivateSubnetAZ1, !Ref PrivateSubnetAZ2, !Ref PrivateSubnetAZ3]]
    Export:
      Name: !Sub '${EnvironmentName}-PrivateSubnets'

  DataSubnets:
    Description: List of data subnet IDs
    Value: !Join [',', [!Ref DataSubnetAZ1, !Ref DataSubnetAZ2, !Ref DataSubnetAZ3]]
    Export:
      Name: !Sub '${EnvironmentName}-DataSubnets'
```

#### Deployment Script

```bash
#!/bin/bash
# deploy-multi-az.sh - Deploy Multi-AZ infrastructure

set -euo pipefail

ENVIRONMENT=${1:-production}
REGION=${2:-eu-south-1}
STACK_NAME="ai-support-vpc-${ENVIRONMENT}"

echo "🚀 Deploying Multi-AZ VPC for environment: ${ENVIRONMENT}"
echo "📍 Region: ${REGION}"

# Validate template
echo "✓ Validating CloudFormation template..."
aws cloudformation validate-template \
  --template-body file://multi-az-vpc.yaml \
  --region "${REGION}"

# Create change set
echo "✓ Creating change set..."
aws cloudformation create-change-set \
  --stack-name "${STACK_NAME}" \
  --change-set-name "deploy-$(date +%s)" \
  --template-body file://multi-az-vpc.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue="${ENVIRONMENT}" \
  --capabilities CAPABILITY_IAM \
  --region "${REGION}"

# Wait for change set creation
echo "⏳ Waiting for change set creation..."
aws cloudformation wait change-set-create-complete \
  --stack-name "${STACK_NAME}" \
  --change-set-name "deploy-$(date +%s)" \
  --region "${REGION}"

# Review changes
echo "📋 Change set details:"
aws cloudformation describe-change-set \
  --stack-name "${STACK_NAME}" \
  --change-set-name "deploy-$(date +%s)" \
  --region "${REGION}" \
  --query 'Changes[].{Action:ResourceChange.Action,Resource:ResourceChange.LogicalResourceId,Type:ResourceChange.ResourceType}' \
  --output table

# Execute change set
read -p "Execute change set? (yes/no): " confirm
if [ "$confirm" = "yes" ]; then
  aws cloudformation execute-change-set \
    --stack-name "${STACK_NAME}" \
    --change-set-name "deploy-$(date +%s)" \
    --region "${REGION}"

  echo "⏳ Waiting for stack deployment..."
  aws cloudformation wait stack-create-complete \
    --stack-name "${STACK_NAME}" \
    --region "${REGION}"

  echo "✅ Multi-AZ VPC deployed successfully!"
else
  echo "❌ Deployment cancelled"
  exit 1
fi

# Output stack details
echo "📊 Stack Outputs:"
aws cloudformation describe-stacks \
  --stack-name "${STACK_NAME}" \
  --region "${REGION}" \
  --query 'Stacks[0].Outputs' \
  --output table
```

### Esempio 2: RDS Multi-AZ Failover Test

Implementiamo RDS PostgreSQL con Multi-AZ e testiamo il failover automatico.

#### CloudFormation - RDS Multi-AZ

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'RDS PostgreSQL Multi-AZ with automated failover'

Parameters:
  EnvironmentName:
    Type: String
    Default: production

  DBInstanceClass:
    Type: String
    Default: db.t3.medium
    Description: Database instance type

  MasterUsername:
    Type: String
    Default: postgres
    NoEcho: false

  MasterUserPassword:
    Type: String
    NoEcho: true
    Description: Master user password (min 8 chars)
    MinLength: 8

Resources:
  # DB Subnet Group (spans 3 AZs)
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupName: !Sub '${EnvironmentName}-db-subnet-group'
      DBSubnetGroupDescription: Subnet group for RDS Multi-AZ
      SubnetIds:
        - !ImportValue
          Fn::Sub: '${EnvironmentName}-DataSubnet-AZ1'
        - !ImportValue
          Fn::Sub: '${EnvironmentName}-DataSubnet-AZ2'
        - !ImportValue
          Fn::Sub: '${EnvironmentName}-DataSubnet-AZ3'
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-db-subnet-group'

  # Security Group for RDS
  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for RDS PostgreSQL
      VpcId: !ImportValue
        Fn::Sub: '${EnvironmentName}-VPC-ID'
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !ImportValue
            Fn::Sub: '${EnvironmentName}-Lambda-SG'
          Description: Allow Lambda functions to connect
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-rds-sg'

  # RDS Parameter Group
  DBParameterGroup:
    Type: AWS::RDS::DBParameterGroup
    Properties:
      Description: Custom parameter group for PostgreSQL
      Family: postgres15
      Parameters:
        shared_preload_libraries: pg_stat_statements
        log_statement: all
        log_min_duration_statement: 1000  # Log queries > 1s
        max_connections: 100
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-pg-params'

  # RDS Instance - Multi-AZ enabled
  DBInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    UpdateReplacePolicy: Snapshot
    Properties:
      DBInstanceIdentifier: !Sub '${EnvironmentName}-analytics-db'
      Engine: postgres
      EngineVersion: '15.4'
      DBInstanceClass: !Ref DBInstanceClass
      AllocatedStorage: 100
      StorageType: gp3
      StorageEncrypted: true
      KmsKeyId: !GetAtt DBEncryptionKey.Arn

      # Multi-AZ configuration
      MultiAZ: true  # 🔑 Critical: Enables automatic failover
      AvailabilityZone: !Select [0, !GetAZs '']  # Primary AZ

      # Network
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref DBSecurityGroup
      PubliclyAccessible: false

      # Credentials
      MasterUsername: !Ref MasterUsername
      MasterUserPassword: !Ref MasterUserPassword

      # Configuration
      DBParameterGroupName: !Ref DBParameterGroup

      # Backup
      BackupRetentionPeriod: 7
      PreferredBackupWindow: '03:00-04:00'  # 3-4 AM UTC
      PreferredMaintenanceWindow: 'sun:04:00-sun:05:00'

      # Monitoring
      EnableCloudwatchLogsExports:
        - postgresql
        - upgrade
      MonitoringInterval: 60  # Enhanced monitoring every 60s
      MonitoringRoleArn: !GetAtt RDSMonitoringRole.Arn
      EnablePerformanceInsights: true
      PerformanceInsightsRetentionPeriod: 7

      # Deletion protection
      DeletionProtection: true

      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-analytics-db'
        - Key: Backup
          Value: 'true'

  # KMS Key for RDS encryption
  DBEncryptionKey:
    Type: AWS::KMS::Key
    Properties:
      Description: KMS key for RDS encryption
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          - Sid: Allow RDS to use the key
            Effect: Allow
            Principal:
              Service: rds.amazonaws.com
            Action:
              - 'kms:Decrypt'
              - 'kms:GenerateDataKey'
              - 'kms:CreateGrant'
            Resource: '*'

  DBEncryptionKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: !Sub 'alias/${EnvironmentName}-rds-key'
      TargetKeyId: !Ref DBEncryptionKey

  # IAM Role for RDS Enhanced Monitoring
  RDSMonitoringRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: monitoring.rds.amazonaws.com
            Action: 'sts:AssumeRole'
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole'
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-rds-monitoring-role'

  # CloudWatch Alarms
  HighCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-rds-high-cpu'
      AlarmDescription: Alert when RDS CPU > 80%
      MetricName: CPUUtilization
      Namespace: AWS/RDS
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: DBInstanceIdentifier
          Value: !Ref DBInstance
      AlarmActions:
        - !Ref SNSTopic

  LowStorageAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-rds-low-storage'
      AlarmDescription: Alert when RDS free storage < 10GB
      MetricName: FreeStorageSpace
      Namespace: AWS/RDS
      Statistic: Average
      Period: 300
      EvaluationPeriods: 1
      Threshold: 10737418240  # 10 GB in bytes
      ComparisonOperator: LessThanThreshold
      Dimensions:
        - Name: DBInstanceIdentifier
          Value: !Ref DBInstance
      AlarmActions:
        - !Ref SNSTopic

  FailoverEventRule:
    Type: AWS::Events::Rule
    Properties:
      Name: !Sub '${EnvironmentName}-rds-failover-event'
      Description: Trigger on RDS failover event
      EventPattern:
        source:
          - aws.rds
        detail-type:
          - RDS DB Instance Event
        detail:
          EventCategories:
            - failover
          SourceIdentifier:
            - !Ref DBInstance
      State: ENABLED
      Targets:
        - Arn: !Ref SNSTopic
          Id: NotifyOnFailover

  SNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      DisplayName: RDS Alerts
      Subscription:
        - Endpoint: ops-team@example.com
          Protocol: email

Outputs:
  DBEndpoint:
    Description: RDS endpoint address
    Value: !GetAtt DBInstance.Endpoint.Address
    Export:
      Name: !Sub '${EnvironmentName}-RDS-Endpoint'

  DBPort:
    Description: RDS endpoint port
    Value: !GetAtt DBInstance.Endpoint.Port
    Export:
      Name: !Sub '${EnvironmentName}-RDS-Port'
```

#### Python Script - Test RDS Failover

```python
#!/usr/bin/env python3
"""
test_rds_failover.py - Automated RDS Multi-AZ failover testing

This script:
1. Connects to RDS and records current endpoint
2. Triggers a manual failover via AWS API
3. Monitors failover progress
4. Measures RTO (Recovery Time Objective)
5. Validates data consistency post-failover
"""

import boto3
import psycopg2
import time
from datetime import datetime, timedelta
from typing import Dict, Tuple

# Configuration
DB_INSTANCE_ID = "production-analytics-db"
REGION = "eu-south-1"
DB_NAME = "analytics"
DB_USER = "postgres"
DB_PASSWORD = "your-password-here"  # Use Secrets Manager in production

# AWS clients
rds_client = boto3.client('rds', region_name=REGION)
cloudwatch = boto3.client('cloudwatch', region_name=REGION)


def get_db_connection(endpoint: str) -> psycopg2.extensions.connection:
    """Create database connection."""
    return psycopg2.connect(
        host=endpoint,
        port=5432,
        database=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD,
        connect_timeout=5
    )


def get_current_endpoint() -> str:
    """Get current RDS endpoint address."""
    response = rds_client.describe_db_instances(
        DBInstanceIdentifier=DB_INSTANCE_ID
    )
    return response['DBInstances'][0]['Endpoint']['Address']


def get_instance_status() -> Dict:
    """Get detailed instance status."""
    response = rds_client.describe_db_instances(
        DBInstanceIdentifier=DB_INSTANCE_ID
    )
    instance = response['DBInstances'][0]
    return {
        'status': instance['DBInstanceStatus'],
        'multi_az': instance['MultiAZ'],
        'availability_zone': instance['AvailabilityZone'],
        'secondary_zone': instance.get('SecondaryAvailabilityZone', 'N/A'),
        'endpoint': instance['Endpoint']['Address']
    }


def create_test_data(conn: psycopg2.extensions.connection) -> int:
    """Create test data and return record ID."""
    with conn.cursor() as cur:
        cur.execute("""
            CREATE TABLE IF NOT EXISTS failover_test (
                id SERIAL PRIMARY KEY,
                test_time TIMESTAMP NOT NULL,
                test_data VARCHAR(255)
            )
        """)

        test_time = datetime.utcnow()
        test_data = f"Failover test started at {test_time}"

        cur.execute(
            "INSERT INTO failover_test (test_time, test_data) VALUES (%s, %s) RETURNING id",
            (test_time, test_data)
        )
        record_id = cur.fetchone()[0]
        conn.commit()

    print(f"✓ Created test record ID: {record_id}")
    return record_id


def verify_test_data(conn: psycopg2.extensions.connection, record_id: int) -> bool:
    """Verify test data exists after failover."""
    with conn.cursor() as cur:
        cur.execute(
            "SELECT id, test_time, test_data FROM failover_test WHERE id = %s",
            (record_id,)
        )
        result = cur.fetchone()

        if result:
            print(f"✓ Data verified - ID: {result[0]}, Time: {result[1]}, Data: {result[2]}")
            return True
        else:
            print("✗ Data verification failed - record not found!")
            return False


def trigger_failover() -> datetime:
    """Trigger manual failover and return start time."""
    print("\n🔄 Triggering manual failover...")

    failover_start = datetime.utcnow()

    response = rds_client.reboot_db_instance(
        DBInstanceIdentifier=DB_INSTANCE_ID,
        ForceFailover=True  # Force failover to standby
    )

    print(f"✓ Failover initiated at {failover_start}")
    return failover_start


def wait_for_available(timeout: int = 600) -> Tuple[bool, float]:
    """
    Wait for instance to become available after failover.

    Returns:
        (success: bool, elapsed_seconds: float)
    """
    print("\n⏳ Waiting for instance to become available...")

    start_time = time.time()

    while time.time() - start_time < timeout:
        status_info = get_instance_status()
        status = status_info['status']

        print(f"  Status: {status} | AZ: {status_info['availability_zone']} | "
              f"Elapsed: {time.time() - start_time:.1f}s")

        if status == 'available':
            elapsed = time.time() - start_time
            print(f"\n✅ Instance available! RTO: {elapsed:.2f} seconds ({elapsed/60:.2f} minutes)")
            return True, elapsed

        time.sleep(10)  # Check every 10 seconds

    print(f"\n❌ Timeout waiting for instance (>{timeout}s)")
    return False, timeout


def test_connectivity_during_failover(endpoint: str, duration: int = 300):
    """
    Continuously test connectivity during failover.

    This demonstrates the actual downtime experienced by applications.
    """
    print(f"\n🔌 Testing connectivity for {duration}s...")

    start_time = time.time()
    downtime_start = None
    downtime_end = None
    connection_attempts = 0
    failed_attempts = 0

    while time.time() - start_time < duration:
        connection_attempts += 1

        try:
            conn = get_db_connection(endpoint)
            with conn.cursor() as cur:
                cur.execute("SELECT 1")
                cur.fetchone()
            conn.close()

            # Connection successful
            if downtime_start and not downtime_end:
                # Downtime just ended
                downtime_end = time.time()
                actual_downtime = downtime_end - downtime_start
                print(f"  ✅ Connection restored! Actual downtime: {actual_downtime:.2f}s")

            print(f"  ✓ Connection OK (attempt {connection_attempts})")

        except Exception as e:
            failed_attempts += 1

            if not downtime_start:
                # Downtime just started
                downtime_start = time.time()
                print(f"  ⚠️  Downtime started at {downtime_start}")

            print(f"  ✗ Connection failed: {str(e)[:50]}...")

        time.sleep(5)  # Test every 5 seconds

    # Summary
    print("\n📊 Connectivity Test Summary:")
    print(f"  Total attempts: {connection_attempts}")
    print(f"  Failed attempts: {failed_attempts}")
    print(f"  Success rate: {((connection_attempts - failed_attempts) / connection_attempts * 100):.1f}%")

    if downtime_start and downtime_end:
        actual_downtime = downtime_end - downtime_start
        print(f"  Measured downtime: {actual_downtime:.2f}s ({actual_downtime/60:.2f} minutes)")


def publish_metrics(metric_name: str, value: float):
    """Publish custom metric to CloudWatch."""
    cloudwatch.put_metric_data(
        Namespace='RDS/FailoverTesting',
        MetricData=[
            {
                'MetricName': metric_name,
                'Value': value,
                'Unit': 'Seconds',
                'Timestamp': datetime.utcnow(),
                'Dimensions': [
                    {
                        'Name': 'DBInstanceIdentifier',
                        'Value': DB_INSTANCE_ID
                    }
                ]
            }
        ]
    )


def main():
    """Execute failover test."""
    print("=" * 60)
    print("RDS Multi-AZ Failover Test")
    print("=" * 60)

    # 1. Pre-failover status
    print("\n1️⃣  Pre-Failover Status")
    status = get_instance_status()
    print(f"  Status: {status['status']}")
    print(f"  Multi-AZ: {status['multi_az']}")
    print(f"  Primary AZ: {status['availability_zone']}")
    print(f"  Secondary AZ: {status['secondary_zone']}")
    print(f"  Endpoint: {status['endpoint']}")

    endpoint = status['endpoint']

    # 2. Create test data
    print("\n2️⃣  Creating Test Data")
    conn = get_db_connection(endpoint)
    test_record_id = create_test_data(conn)
    conn.close()

    # 3. Trigger failover
    print("\n3️⃣  Initiating Failover")
    failover_start_time = trigger_failover()

    # 4. Wait for completion
    print("\n4️⃣  Monitoring Failover Progress")
    success, rto = wait_for_available(timeout=600)

    if not success:
        print("\n❌ Failover test FAILED - timeout")
        return

    # 5. Post-failover status
    print("\n5️⃣  Post-Failover Status")
    new_status = get_instance_status()
    print(f"  Status: {new_status['status']}")
    print(f"  New Primary AZ: {new_status['availability_zone']}")
    print(f"  Endpoint: {new_status['endpoint']}")

    # Verify AZ changed
    if new_status['availability_zone'] != status['availability_zone']:
        print(f"  ✓ Failover confirmed: {status['availability_zone']} → {new_status['availability_zone']}")
    else:
        print("  ⚠️  Warning: AZ did not change")

    # 6. Verify data integrity
    print("\n6️⃣  Verifying Data Integrity")
    conn = get_db_connection(endpoint)
    data_ok = verify_test_data(conn, test_record_id)
    conn.close()

    # 7. Publish metrics
    print("\n7️⃣  Publishing Metrics to CloudWatch")
    publish_metrics('FailoverRTO', rto)
    print(f"  ✓ Metric published: RTO = {rto:.2f}s")

    # 8. Summary
    print("\n" + "=" * 60)
    print("FAILOVER TEST SUMMARY")
    print("=" * 60)
    print(f"✅ Test Status: {'PASSED' if data_ok else 'FAILED'}")
    print(f"⏱️  RTO (Recovery Time): {rto:.2f}s ({rto/60:.2f} minutes)")
    print(f"✓ RPO (Data Loss): 0 seconds (no data loss)")
    print(f"🔄 AZ Change: {status['availability_zone']} → {new_status['availability_zone']}")
    print(f"📊 Data Integrity: {'OK' if data_ok else 'FAILED'}")
    print("=" * 60)


if __name__ == '__main__':
    main()
```

#### Expected Results

When running the failover test, you should see:

```
==============================================================
RDS Multi-AZ Failover Test
==============================================================

1️⃣  Pre-Failover Status
  Status: available
  Multi-AZ: True
  Primary AZ: eu-south-1a
  Secondary AZ: eu-south-1b
  Endpoint: production-analytics-db.xxxxx.eu-south-1.rds.amazonaws.com

2️⃣  Creating Test Data
✓ Created test record ID: 42

3️⃣  Initiating Failover
🔄 Triggering manual failover...
✓ Failover initiated at 2025-11-18 14:30:00

4️⃣  Monitoring Failover Progress
⏳ Waiting for instance to become available...
  Status: rebooting | AZ: eu-south-1a | Elapsed: 10.2s
  Status: rebooting | AZ: eu-south-1a | Elapsed: 20.5s
  Status: available | AZ: eu-south-1b | Elapsed: 127.8s

✅ Instance available! RTO: 127.8 seconds (2.13 minutes)

5️⃣  Post-Failover Status
  Status: available
  New Primary AZ: eu-south-1b
  Endpoint: production-analytics-db.xxxxx.eu-south-1.rds.amazonaws.com
  ✓ Failover confirmed: eu-south-1a → eu-south-1b

6️⃣  Verifying Data Integrity
✓ Data verified - ID: 42, Time: 2025-11-18 14:29:55, Data: Failover test started...

7️⃣  Publishing Metrics to CloudWatch
  ✓ Metric published: RTO = 127.8s

==============================================================
FAILOVER TEST SUMMARY
==============================================================
✅ Test Status: PASSED
⏱️  RTO (Recovery Time): 127.8s (2.13 minutes)
✓ RPO (Data Loss): 0 seconds (no data loss)
🔄 AZ Change: eu-south-1a → eu-south-1b
📊 Data Integrity: OK
==============================================================
```

### Esempio 3: S3 Cross-Region Replication

Configuriamo la replica cross-region di S3 per disaster recovery.

#### CloudFormation - S3 Buckets con CRR

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'S3 buckets with Cross-Region Replication for DR'

Parameters:
  EnvironmentName:
    Type: String
    Default: production

  PrimaryRegion:
    Type: String
    Default: eu-south-1

  ReplicaRegion:
    Type: String
    Default: eu-west-1

Resources:
  # Primary bucket (source)
  PrimaryBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'ai-support-kb-${EnvironmentName}-${AWS::AccountId}-primary'

      # Versioning required for replication
      VersioningConfiguration:
        Status: Enabled

      # Encryption
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !GetAtt PrimaryKMSKey.Arn
            BucketKeyEnabled: true

      # Replication configuration
      ReplicationConfiguration:
        Role: !GetAtt ReplicationRole.Arn
        Rules:
          - Id: ReplicateAll
            Status: Enabled
            Priority: 1
            Filter:
              Prefix: ''  # Replicate all objects
            Destination:
              Bucket: !Sub 'arn:aws:s3:::ai-support-kb-${EnvironmentName}-${AWS::AccountId}-replica'
              ReplicationTime:
                Status: Enabled
                Time:
                  Minutes: 15  # S3 RTC: replicate within 15 minutes
              Metrics:
                Status: Enabled
                EventThreshold:
                  Minutes: 15
              # Encryption in destination
              EncryptionConfiguration:
                ReplicaKmsKeyID: !Sub 'arn:aws:kms:${ReplicaRegion}:${AWS::AccountId}:alias/${EnvironmentName}-replica-key'
            DeleteMarkerReplication:
              Status: Enabled  # Replicate deletions
            SourceSelectionCriteria:
              SseKmsEncryptedObjects:
                Status: Enabled  # Replicate encrypted objects

      # Lifecycle policies
      LifecycleConfiguration:
        Rules:
          - Id: TransitionOldVersions
            Status: Enabled
            NoncurrentVersionTransitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
              - TransitionInDays: 90
                StorageClass: GLACIER
            NoncurrentVersionExpiration:
              NoncurrentDays: 365

      # Access logging
      LoggingConfiguration:
        DestinationBucketName: !Ref LoggingBucket
        LogFilePrefix: primary-bucket-logs/

      # Public access block
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-primary-kb'
        - Key: ReplicationEnabled
          Value: 'true'

  # Replica bucket (destination) - must be created in replica region
  # Note: This requires a separate stack deployment in eu-west-1

  # KMS Key for primary bucket
  PrimaryKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: KMS key for S3 bucket encryption (primary)
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'

          - Sid: Allow S3 to use the key
            Effect: Allow
            Principal:
              Service: s3.amazonaws.com
            Action:
              - 'kms:Decrypt'
              - 'kms:GenerateDataKey'
            Resource: '*'

          - Sid: Allow Replication Role
            Effect: Allow
            Principal:
              AWS: !GetAtt ReplicationRole.Arn
            Action:
              - 'kms:Decrypt'
              - 'kms:DescribeKey'
            Resource: '*'

  PrimaryKMSKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: !Sub 'alias/${EnvironmentName}-primary-key'
      TargetKeyId: !Ref PrimaryKMSKey

  # IAM Role for replication
  ReplicationRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: s3.amazonaws.com
            Action: 'sts:AssumeRole'
      Policies:
        - PolicyName: S3ReplicationPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              # Source bucket permissions
              - Effect: Allow
                Action:
                  - 's3:GetReplicationConfiguration'
                  - 's3:ListBucket'
                Resource: !GetAtt PrimaryBucket.Arn

              - Effect: Allow
                Action:
                  - 's3:GetObjectVersionForReplication'
                  - 's3:GetObjectVersionAcl'
                  - 's3:GetObjectVersionTagging'
                Resource: !Sub '${PrimaryBucket.Arn}/*'

              # Destination bucket permissions
              - Effect: Allow
                Action:
                  - 's3:ReplicateObject'
                  - 's3:ReplicateDelete'
                  - 's3:ReplicateTags'
                Resource: !Sub 'arn:aws:s3:::ai-support-kb-${EnvironmentName}-${AWS::AccountId}-replica/*'

              # KMS permissions
              - Effect: Allow
                Action:
                  - 'kms:Decrypt'
                Resource: !GetAtt PrimaryKMSKey.Arn

              - Effect: Allow
                Action:
                  - 'kms:Encrypt'
                Resource: !Sub 'arn:aws:kms:${ReplicaRegion}:${AWS::AccountId}:alias/${EnvironmentName}-replica-key'
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-s3-replication-role'

  # Logging bucket
  LoggingBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'ai-support-logs-${EnvironmentName}-${AWS::AccountId}'
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      LifecycleConfiguration:
        Rules:
          - Id: ExpireOldLogs
            Status: Enabled
            ExpirationInDays: 90

  # CloudWatch Alarm for replication failures
  ReplicationFailureAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-s3-replication-failure'
      AlarmDescription: Alert when S3 replication has failures
      MetricName: ReplicationLatency
      Namespace: AWS/S3
      Statistic: Maximum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 900000  # 15 minutes in milliseconds
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: SourceBucket
          Value: !Ref PrimaryBucket
        - Name: DestinationBucket
          Value: !Sub 'ai-support-kb-${EnvironmentName}-${AWS::AccountId}-replica'
        - Name: RuleId
          Value: ReplicateAll
      TreatMissingData: notBreaching

Outputs:
  PrimaryBucketName:
    Description: Primary S3 bucket name
    Value: !Ref PrimaryBucket
    Export:
      Name: !Sub '${EnvironmentName}-PrimaryBucket'

  PrimaryBucketArn:
    Description: Primary S3 bucket ARN
    Value: !GetAtt PrimaryBucket.Arn

  ReplicationRoleArn:
    Description: Replication IAM role ARN
    Value: !GetAtt ReplicationRole.Arn
```

#### Python Script - Monitor Replication Status

```python
#!/usr/bin/env python3
"""
monitor_s3_replication.py - Monitor S3 Cross-Region Replication status

Monitors replication metrics and validates data consistency.
"""

import boto3
from datetime import datetime, timedelta
from typing import Dict, List
import json

# Configuration
PRIMARY_BUCKET = "ai-support-kb-production-123456789-primary"
REPLICA_BUCKET = "ai-support-kb-production-123456789-replica"
PRIMARY_REGION = "eu-south-1"
REPLICA_REGION = "eu-west-1"

# AWS clients
s3_primary = boto3.client('s3', region_name=PRIMARY_REGION)
s3_replica = boto3.client('s3', region_name=REPLICA_REGION)
cloudwatch = boto3.client('cloudwatch', region_name=PRIMARY_REGION)


def get_replication_metrics() -> Dict:
    """Fetch S3 replication metrics from CloudWatch."""
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=1)

    # Bytes Pending Replication
    bytes_pending = cloudwatch.get_metric_statistics(
        Namespace='AWS/S3',
        MetricName='BytesPendingReplication',
        Dimensions=[
            {'Name': 'SourceBucket', 'Value': PRIMARY_BUCKET},
            {'Name': 'DestinationBucket', 'Value': REPLICA_BUCKET},
            {'Name': 'RuleId', 'Value': 'ReplicateAll'}
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average', 'Maximum']
    )

    # Replication Latency
    latency = cloudwatch.get_metric_statistics(
        Namespace='AWS/S3',
        MetricName='ReplicationLatency',
        Dimensions=[
            {'Name': 'SourceBucket', 'Value': PRIMARY_BUCKET},
            {'Name': 'DestinationBucket', 'Value': REPLICA_BUCKET},
            {'Name': 'RuleId', 'Value': 'ReplicateAll'}
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average', 'Maximum']
    )

    # Operations Pending
    ops_pending = cloudwatch.get_metric_statistics(
        Namespace='AWS/S3',
        MetricName='OperationsPendingReplication',
        Dimensions=[
            {'Name': 'SourceBucket', 'Value': PRIMARY_BUCKET},
            {'Name': 'DestinationBucket', 'Value': REPLICA_BUCKET},
            {'Name': 'RuleId', 'Value': 'ReplicateAll'}
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average', 'Maximum']
    )

    return {
        'bytes_pending': bytes_pending['Datapoints'],
        'latency': latency['Datapoints'],
        'operations_pending': ops_pending['Datapoints']
    }


def test_replication_consistency(num_samples: int = 10):
    """
    Test data consistency between primary and replica.

    Samples random objects and verifies they exist in replica with same ETag.
    """
    print(f"\n🔍 Testing replication consistency ({num_samples} samples)...")

    # List objects in primary bucket
    response = s3_primary.list_objects_v2(
        Bucket=PRIMARY_BUCKET,
        MaxKeys=num_samples * 2  # Get more than needed for sampling
    )

    if 'Contents' not in response or len(response['Contents']) == 0:
        print("  ⚠️  No objects found in primary bucket")
        return

    objects = response['Contents'][:num_samples]

    consistent = 0
    inconsistent = 0
    missing = 0

    for obj in objects:
        key = obj['Key']
        primary_etag = obj['ETag']

        try:
            # Check if object exists in replica
            replica_obj = s3_replica.head_object(
                Bucket=REPLICA_BUCKET,
                Key=key
            )
            replica_etag = replica_obj['ETag']

            # Check replication status
            replication_status = replica_obj.get('ReplicationStatus', 'UNKNOWN')

            if primary_etag == replica_etag:
                consistent += 1
                print(f"  ✓ {key}: CONSISTENT (status: {replication_status})")
            else:
                inconsistent += 1
                print(f"  ✗ {key}: INCONSISTENT (primary: {primary_etag}, replica: {replica_etag})")

        except s3_replica.exceptions.NoSuchKey:
            missing += 1
            print(f"  ⚠️  {key}: MISSING in replica")

        except Exception as e:
            print(f"  ✗ {key}: ERROR - {str(e)}")

    # Summary
    total = consistent + inconsistent + missing
    print(f"\n📊 Consistency Results:")
    print(f"  Consistent: {consistent}/{total} ({consistent/total*100:.1f}%)")
    print(f"  Inconsistent: {inconsistent}/{total}")
    print(f"  Missing: {missing}/{total}")

    return {
        'consistent': consistent,
        'inconsistent': inconsistent,
        'missing': missing,
        'total': total
    }


def upload_test_object() -> Dict:
    """Upload test object and measure replication time."""
    test_key = f"replication-test/{datetime.utcnow().isoformat()}.txt"
    test_content = f"Replication test at {datetime.utcnow()}"

    print(f"\n📤 Uploading test object: {test_key}")

    # Upload to primary
    upload_time = datetime.utcnow()
    s3_primary.put_object(
        Bucket=PRIMARY_BUCKET,
        Key=test_key,
        Body=test_content.encode('utf-8'),
        ContentType='text/plain'
    )

    print(f"  ✓ Uploaded to primary at {upload_time}")

    # Wait and check replica
    import time
    max_wait = 300  # 5 minutes
    check_interval = 5  # 5 seconds
    elapsed = 0

    print("  ⏳ Waiting for replication...")

    while elapsed < max_wait:
        time.sleep(check_interval)
        elapsed += check_interval

        try:
            replica_obj = s3_replica.head_object(
                Bucket=REPLICA_BUCKET,
                Key=test_key
            )

            replication_time = datetime.utcnow()
            latency = (replication_time - upload_time).total_seconds()

            print(f"  ✅ Replicated in {latency:.1f}s (status: {replica_obj.get('ReplicationStatus', 'UNKNOWN')})")

            return {
                'success': True,
                'latency_seconds': latency,
                'key': test_key
            }

        except s3_replica.exceptions.NoSuchKey:
            print(f"    Waiting... ({elapsed}s elapsed)")
            continue

    print(f"  ❌ Replication timeout after {max_wait}s")
    return {
        'success': False,
        'latency_seconds': max_wait,
        'key': test_key
    }


def main():
    """Execute replication monitoring."""
    print("=" * 60)
    print("S3 Cross-Region Replication Monitor")
    print("=" * 60)
    print(f"Primary: {PRIMARY_BUCKET} ({PRIMARY_REGION})")
    print(f"Replica: {REPLICA_BUCKET} ({REPLICA_REGION})")

    # 1. Check replication metrics
    print("\n1️⃣  Replication Metrics (last hour)")
    metrics = get_replication_metrics()

    if metrics['bytes_pending']:
        latest_bytes = max(metrics['bytes_pending'], key=lambda x: x['Timestamp'])
        print(f"  Bytes Pending: {latest_bytes.get('Average', 0):,.0f} bytes")
    else:
        print("  Bytes Pending: 0 (no pending data)")

    if metrics['latency']:
        latest_latency = max(metrics['latency'], key=lambda x: x['Timestamp'])
        avg_latency_ms = latest_latency.get('Average', 0)
        max_latency_ms = latest_latency.get('Maximum', 0)
        print(f"  Avg Latency: {avg_latency_ms/1000:.1f}s")
        print(f"  Max Latency: {max_latency_ms/1000:.1f}s")
    else:
        print("  Latency: No data")

    if metrics['operations_pending']:
        latest_ops = max(metrics['operations_pending'], key=lambda x: x['Timestamp'])
        print(f"  Operations Pending: {latest_ops.get('Average', 0):.0f}")
    else:
        print("  Operations Pending: 0")

    # 2. Test consistency
    print("\n2️⃣  Data Consistency Check")
    consistency = test_replication_consistency(num_samples=10)

    # 3. Live replication test
    print("\n3️⃣  Live Replication Test")
    test_result = upload_test_object()

    # Summary
    print("\n" + "=" * 60)
    print("SUMMARY")
    print("=" * 60)

    if consistency:
        consistency_pct = (consistency['consistent'] / consistency['total'] * 100)
        print(f"✓ Consistency: {consistency_pct:.1f}% ({consistency['consistent']}/{consistency['total']})")

    if test_result['success']:
        print(f"✓ Live Test: PASSED (latency: {test_result['latency_seconds']:.1f}s)")

        # Meets S3 RTC SLA?
        if test_result['latency_seconds'] <= 900:  # 15 minutes
            print("  ✅ Within S3 Replication Time Control SLA (15min)")
        else:
            print("  ⚠️  Exceeds S3 RTC SLA")
    else:
        print(f"✗ Live Test: FAILED (timeout)")

    print("=" * 60)


if __name__ == '__main__':
    main()
```

### Esempio 4: DynamoDB Global Tables (Multi-Region Active-Active)

Configuriamo DynamoDB Global Tables per active-active cross-region replication.

#### CloudFormation - DynamoDB Global Table

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'DynamoDB Global Table for multi-region active-active deployment'

Parameters:
  EnvironmentName:
    Type: String
    Default: production

  TableName:
    Type: String
    Default: ai-support-tickets
    Description: Table name (must be unique globally)

Resources:
  # DynamoDB Global Table
  GlobalTable:
    Type: AWS::DynamoDB::GlobalTable
    Properties:
      TableName: !Sub '${EnvironmentName}-${TableName}'

      # Attribute definitions
      AttributeDefinitions:
        - AttributeName: ticket_id
          AttributeType: S
        - AttributeName: customer_id
          AttributeType: S
        - AttributeName: created_at
          AttributeType: S
        - AttributeName: status
          AttributeType: S

      # Key schema
      KeySchema:
        - AttributeName: ticket_id
          KeyType: HASH  # Partition key
        - AttributeName: created_at
          KeyType: RANGE  # Sort key

      # Billing mode
      BillingMode: PAY_PER_REQUEST  # On-demand for unpredictable workloads

      # Stream specification (required for global tables)
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES

      # SSE encryption
      SSESpecification:
        SSEEnabled: true
        SSEType: KMS
        # Note: Each region uses its own CMK for global tables

      # Time to Live
      TimeToLiveSpecification:
        Enabled: true
        AttributeName: ttl

      # Replicas (regions for global table)
      Replicas:
        # Primary region: eu-south-1 (Milano)
        - Region: eu-south-1
          GlobalSecondaryIndexes:
            - IndexName: customer-index
              KeySchema:
                - AttributeName: customer_id
                  KeyType: HASH
                - AttributeName: created_at
                  KeyType: RANGE
              Projection:
                ProjectionType: ALL

            - IndexName: status-index
              KeySchema:
                - AttributeName: status
                  KeyType: HASH
                - AttributeName: created_at
                  KeyType: RANGE
              Projection:
                ProjectionType: ALL

          PointInTimeRecoverySpecification:
            PointInTimeRecoveryEnabled: true

          TableClass: STANDARD

          Tags:
            - Key: Name
              Value: !Sub '${EnvironmentName}-tickets-primary'
            - Key: Region
              Value: eu-south-1

        # Replica region: eu-west-1 (Irlanda)
        - Region: eu-west-1
          GlobalSecondaryIndexes:
            - IndexName: customer-index
              KeySchema:
                - AttributeName: customer_id
                  KeyType: HASH
                - AttributeName: created_at
                  KeyType: RANGE
              Projection:
                ProjectionType: ALL

            - IndexName: status-index
              KeySchema:
                - AttributeName: status
                  KeyType: HASH
                - AttributeName: created_at
                  KeyType: RANGE
              Projection:
                ProjectionType: ALL

          PointInTimeRecoverySpecification:
            PointInTimeRecoveryEnabled: true

          TableClass: STANDARD

          Tags:
            - Key: Name
              Value: !Sub '${EnvironmentName}-tickets-replica'
            - Key: Region
              Value: eu-west-1

  # CloudWatch Alarm for replication lag
  ReplicationLatencyAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-dynamodb-replication-lag'
      AlarmDescription: Alert when DynamoDB replication lag exceeds threshold
      MetricName: ReplicationLatency
      Namespace: AWS/DynamoDB
      Statistic: Maximum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 10000  # 10 seconds in milliseconds
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: TableName
          Value: !Sub '${EnvironmentName}-${TableName}'
        - Name: ReceivingRegion
          Value: eu-west-1

Outputs:
  TableName:
    Description: Global table name
    Value: !Ref GlobalTable
    Export:
      Name: !Sub '${EnvironmentName}-GlobalTable-Name'

  TableArn:
    Description: Global table ARN
    Value: !GetAtt GlobalTable.Arn

  StreamArn:
    Description: Table stream ARN
    Value: !GetAtt GlobalTable.StreamArn
```

#### Python Script - Test Global Table Replication

```python
#!/usr/bin/env python3
"""
test_dynamodb_global_table.py - Test DynamoDB Global Tables replication

Tests active-active replication, conflict resolution, and failover capabilities.
"""

import boto3
from datetime import datetime, timedelta
import time
import uuid
from typing import Dict, Optional

# Configuration
TABLE_NAME = "production-ai-support-tickets"
PRIMARY_REGION = "eu-south-1"
REPLICA_REGION = "eu-west-1"

# AWS clients
dynamodb_primary = boto3.resource('dynamodb', region_name=PRIMARY_REGION)
dynamodb_replica = boto3.resource('dynamodb', region_name=REPLICA_REGION)
cloudwatch = boto3.client('cloudwatch', region_name=PRIMARY_REGION)

table_primary = dynamodb_primary.Table(TABLE_NAME)
table_replica = dynamodb_replica.Table(TABLE_NAME)


def write_to_primary(ticket_id: str, data: Dict) -> datetime:
    """Write item to primary region."""
    write_time = datetime.utcnow()

    item = {
        'ticket_id': ticket_id,
        'created_at': write_time.isoformat(),
        'customer_id': data.get('customer_id', 'CUST-001'),
        'status': data.get('status', 'open'),
        'description': data.get('description', ''),
        'region_written': PRIMARY_REGION,
        'write_timestamp': int(write_time.timestamp() * 1000)
    }

    table_primary.put_item(Item=item)
    print(f"  ✓ Written to PRIMARY ({PRIMARY_REGION}) at {write_time}")

    return write_time


def write_to_replica(ticket_id: str, data: Dict) -> datetime:
    """Write item to replica region."""
    write_time = datetime.utcnow()

    item = {
        'ticket_id': ticket_id,
        'created_at': write_time.isoformat(),
        'customer_id': data.get('customer_id', 'CUST-001'),
        'status': data.get('status', 'open'),
        'description': data.get('description', ''),
        'region_written': REPLICA_REGION,
        'write_timestamp': int(write_time.timestamp() * 1000)
    }

    table_replica.put_item(Item=item)
    print(f"  ✓ Written to REPLICA ({REPLICA_REGION}) at {write_time}")

    return write_time


def read_from_region(table, ticket_id: str, created_at: str) -> Optional[Dict]:
    """Read item from specific region."""
    try:
        response = table.get_item(
            Key={
                'ticket_id': ticket_id,
                'created_at': created_at
            }
        )
        return response.get('Item')
    except Exception as e:
        print(f"  ✗ Read error: {str(e)}")
        return None


def test_replication_latency():
    """
    Test 1: Measure replication latency

    Write to primary, measure time until visible in replica.
    """
    print("\n📝 Test 1: Replication Latency")
    print("=" * 60)

    ticket_id = f"TICKET-{uuid.uuid4()}"
    created_at = datetime.utcnow().isoformat()

    # Write to primary
    write_time = write_to_primary(ticket_id, {
        'customer_id': 'CUST-LAT-TEST',
        'status': 'open',
        'description': 'Replication latency test'
    })

    # Poll replica until item appears
    max_wait = 60  # 60 seconds
    check_interval = 0.5  # 500ms
    elapsed = 0

    print(f"  ⏳ Polling replica for ticket {ticket_id}...")

    while elapsed < max_wait:
        time.sleep(check_interval)
        elapsed += check_interval

        item = read_from_region(table_replica, ticket_id, created_at)

        if item:
            replication_time = datetime.utcnow()
            latency = (replication_time - write_time).total_seconds()

            print(f"  ✅ Item replicated in {latency:.3f}s")
            print(f"     Region written: {item.get('region_written')}")
            print(f"     Replication lag: {latency*1000:.0f}ms")

            # Publish metric
            cloudwatch.put_metric_data(
                Namespace='DynamoDB/Testing',
                MetricData=[
                    {
                        'MetricName': 'ReplicationLatency',
                        'Value': latency * 1000,  # milliseconds
                        'Unit': 'Milliseconds',
                        'Timestamp': datetime.utcnow()
                    }
                ]
            )

            return latency

    print(f"  ❌ Replication timeout after {max_wait}s")
    return None


def test_bidirectional_replication():
    """
    Test 2: Bi-directional replication

    Write to both regions and verify both items replicate correctly.
    """
    print("\n🔄 Test 2: Bi-directional Replication")
    print("=" * 60)

    ticket_id_1 = f"TICKET-{uuid.uuid4()}"
    ticket_id_2 = f"TICKET-{uuid.uuid4()}"
    created_at = datetime.utcnow().isoformat()

    # Write to primary
    print("\n  Writing to PRIMARY region...")
    write_to_primary(ticket_id_1, {
        'customer_id': 'CUST-BIDIR-1',
        'status': 'open',
        'description': 'Written from primary'
    })

    # Write to replica
    print("\n  Writing to REPLICA region...")
    write_to_replica(ticket_id_2, {
        'customer_id': 'CUST-BIDIR-2',
        'status': 'in_progress',
        'description': 'Written from replica'
    })

    # Wait for replication
    print("\n  ⏳ Waiting 5s for replication...")
    time.sleep(5)

    # Verify item 1 in replica
    print(f"\n  Checking if {ticket_id_1} reached REPLICA...")
    item_1_in_replica = read_from_region(table_replica, ticket_id_1, created_at)
    if item_1_in_replica:
        print(f"    ✅ Found in replica (written from {item_1_in_replica.get('region_written')})")
    else:
        print(f"    ❌ NOT found in replica")

    # Verify item 2 in primary
    print(f"\n  Checking if {ticket_id_2} reached PRIMARY...")
    item_2_in_primary = read_from_region(table_primary, ticket_id_2, created_at)
    if item_2_in_primary:
        print(f"    ✅ Found in primary (written from {item_2_in_primary.get('region_written')})")
    else:
        print(f"    ❌ NOT found in primary")

    success = bool(item_1_in_replica and item_2_in_primary)

    print(f"\n  Result: {'✅ PASSED' if success else '❌ FAILED'}")
    return success


def test_conflict_resolution():
    """
    Test 3: Last-Writer-Wins conflict resolution

    Write same item to both regions simultaneously, verify LWW.
    """
    print("\n⚔️  Test 3: Conflict Resolution (Last-Writer-Wins)")
    print("=" * 60)

    ticket_id = f"TICKET-{uuid.uuid4()}"
    created_at = datetime.utcnow().isoformat()

    # Write conflicting updates
    print("\n  Creating conflicting writes...")

    # Update 1: Primary region (first)
    table_primary.put_item(Item={
        'ticket_id': ticket_id,
        'created_at': created_at,
        'customer_id': 'CUST-CONFLICT',
        'status': 'open',
        'description': 'Written from PRIMARY',
        'region_written': PRIMARY_REGION,
        'write_timestamp': int(datetime.utcnow().timestamp() * 1000),
        'version': 1
    })
    print(f"    ✓ Write 1: PRIMARY (status=open, version=1)")

    time.sleep(0.1)  # Small delay

    # Update 2: Replica region (second, should win)
    table_replica.put_item(Item={
        'ticket_id': ticket_id,
        'created_at': created_at,
        'customer_id': 'CUST-CONFLICT',
        'status': 'resolved',
        'description': 'Written from REPLICA',
        'region_written': REPLICA_REGION,
        'write_timestamp': int(datetime.utcnow().timestamp() * 1000),
        'version': 2
    })
    print(f"    ✓ Write 2: REPLICA (status=resolved, version=2)")

    # Wait for convergence
    print("\n  ⏳ Waiting 10s for conflict resolution...")
    time.sleep(10)

    # Read from both regions
    item_primary = read_from_region(table_primary, ticket_id, created_at)
    item_replica = read_from_region(table_replica, ticket_id, created_at)

    print(f"\n  Final state in PRIMARY:")
    print(f"    Status: {item_primary.get('status')}")
    print(f"    Version: {item_primary.get('version')}")
    print(f"    Region written: {item_primary.get('region_written')}")

    print(f"\n  Final state in REPLICA:")
    print(f"    Status: {item_replica.get('status')}")
    print(f"    Version: {item_replica.get('version')}")
    print(f"    Region written: {item_replica.get('region_written')}")

    # Verify convergence (both should have same final state)
    converged = (
        item_primary.get('status') == item_replica.get('status') and
        item_primary.get('version') == item_replica.get('version')
    )

    # Last writer should win (version 2 from replica)
    correct_winner = (item_primary.get('version') == 2 and
                      item_primary.get('status') == 'resolved')

    print(f"\n  Convergence: {'✅ YES' if converged else '❌ NO'}")
    print(f"  Last-Writer-Wins: {'✅ CORRECT (v2 won)' if correct_winner else '❌ INCORRECT'}")

    return converged and correct_winner


def test_eventual_consistency():
    """
    Test 4: Eventual consistency guarantee

    Write 100 items and verify all eventually appear in both regions.
    """
    print("\n📊 Test 4: Eventual Consistency (100 items)")
    print("=" * 60)

    num_items = 100
    ticket_ids = []
    created_at = datetime.utcnow().isoformat()

    # Write items to primary
    print(f"\n  Writing {num_items} items to PRIMARY...")
    for i in range(num_items):
        ticket_id = f"TICKET-EC-{i:03d}"
        ticket_ids.append(ticket_id)

        table_primary.put_item(Item={
            'ticket_id': ticket_id,
            'created_at': created_at,
            'customer_id': f'CUST-{i:03d}',
            'status': 'open',
            'description': f'Eventual consistency test {i}',
            'region_written': PRIMARY_REGION
        })

    print(f"  ✓ Wrote {num_items} items")

    # Wait and verify in replica
    print(f"\n  ⏳ Waiting 15s for replication...")
    time.sleep(15)

    print(f"\n  Verifying items in REPLICA...")
    found = 0
    missing = []

    for ticket_id in ticket_ids:
        item = read_from_region(table_replica, ticket_id, created_at)
        if item:
            found += 1
        else:
            missing.append(ticket_id)

    consistency_rate = (found / num_items) * 100

    print(f"\n  Results:")
    print(f"    Found: {found}/{num_items} ({consistency_rate:.1f}%)")
    print(f"    Missing: {len(missing)}")

    if missing:
        print(f"    Missing IDs: {', '.join(missing[:10])}{'...' if len(missing) > 10 else ''}")

    success = (consistency_rate == 100.0)
    print(f"\n  Result: {'✅ PASSED (100% consistent)' if success else f'⚠️  PARTIAL ({consistency_rate:.1f}%)'}")

    return success, consistency_rate


def get_replication_metrics():
    """Fetch replication metrics from CloudWatch."""
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=1)

    try:
        response = cloudwatch.get_metric_statistics(
            Namespace='AWS/DynamoDB',
            MetricName='ReplicationLatency',
            Dimensions=[
                {'Name': 'TableName', 'Value': TABLE_NAME},
                {'Name': 'ReceivingRegion', 'Value': REPLICA_REGION}
            ],
            StartTime=start_time,
            EndTime=end_time,
            Period=300,
            Statistics=['Average', 'Maximum']
        )

        return response.get('Datapoints', [])
    except Exception as e:
        print(f"  ⚠️  Could not fetch metrics: {str(e)}")
        return []


def main():
    """Execute all Global Table tests."""
    print("=" * 60)
    print("DynamoDB Global Tables Test Suite")
    print("=" * 60)
    print(f"Table: {TABLE_NAME}")
    print(f"Primary: {PRIMARY_REGION}")
    print(f"Replica: {REPLICA_REGION}")

    results = {}

    # Test 1: Replication Latency
    latency = test_replication_latency()
    results['replication_latency'] = latency

    # Test 2: Bi-directional
    bidir_success = test_bidirectional_replication()
    results['bidirectional'] = bidir_success

    # Test 3: Conflict Resolution
    conflict_success = test_conflict_resolution()
    results['conflict_resolution'] = conflict_success

    # Test 4: Eventual Consistency
    consistency_success, consistency_rate = test_eventual_consistency()
    results['eventual_consistency'] = consistency_success
    results['consistency_rate'] = consistency_rate

    # Fetch CloudWatch metrics
    print("\n📈 CloudWatch Replication Metrics (last hour)")
    print("=" * 60)
    metrics = get_replication_metrics()
    if metrics:
        latest = max(metrics, key=lambda x: x['Timestamp'])
        print(f"  Avg Latency: {latest.get('Average', 0):.1f}ms")
        print(f"  Max Latency: {latest.get('Maximum', 0):.1f}ms")
    else:
        print("  No metrics available")

    # Summary
    print("\n" + "=" * 60)
    print("TEST SUMMARY")
    print("=" * 60)

    if results.get('replication_latency'):
        print(f"✓ Replication Latency: {results['replication_latency']*1000:.0f}ms")
    else:
        print("✗ Replication Latency: FAILED")

    print(f"{'✓' if results.get('bidirectional') else '✗'} Bi-directional Replication: "
          f"{'PASSED' if results.get('bidirectional') else 'FAILED'}")

    print(f"{'✓' if results.get('conflict_resolution') else '✗'} Conflict Resolution: "
          f"{'PASSED' if results.get('conflict_resolution') else 'FAILED'}")

    print(f"{'✓' if results.get('eventual_consistency') else '✗'} Eventual Consistency: "
          f"{results.get('consistency_rate', 0):.1f}%")

    # Overall result
    all_passed = (
        results.get('replication_latency') is not None and
        results.get('bidirectional') and
        results.get('conflict_resolution') and
        results.get('consistency_rate', 0) >= 99.0
    )

    print("\n" + "=" * 60)
    print(f"OVERALL: {'✅ ALL TESTS PASSED' if all_passed else '⚠️  SOME TESTS FAILED'}")
    print("=" * 60)


if __name__ == '__main__':
    main()
```

### Esempio 5: Route 53 Health Check e Failover

Configuriamo Route 53 per failover automatico DNS-based tra regioni.

#### CloudFormation - Route 53 Failover Configuration

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Route 53 health checks and DNS failover'

Parameters:
  DomainName:
    Type: String
    Default: api.ai-support.example.com
    Description: Domain name for the API

  PrimaryEndpoint:
    Type: String
    Default: primary-alb-1234567890.eu-south-1.elb.amazonaws.com
    Description: Primary region ALB endpoint

  SecondaryEndpoint:
    Type: String
    Default: secondary-alb-9876543210.eu-west-1.elb.amazonaws.com
    Description: Secondary region ALB endpoint

Resources:
  # Health check for primary endpoint
  PrimaryHealthCheck:
    Type: AWS::Route53::HealthCheck
    Properties:
      HealthCheckConfig:
        Type: HTTPS
        ResourcePath: /health
        FullyQualifiedDomainName: !Ref PrimaryEndpoint
        Port: 443
        RequestInterval: 30  # Check every 30 seconds
        FailureThreshold: 3  # Mark unhealthy after 3 failures (90s)
        MeasureLatency: true
        EnableSNI: true

      HealthCheckTags:
        - Key: Name
          Value: primary-endpoint-health
        - Key: Region
          Value: eu-south-1

  # Health check for secondary endpoint
  SecondaryHealthCheck:
    Type: AWS::Route53::HealthCheck
    Properties:
      HealthCheckConfig:
        Type: HTTPS
        ResourcePath: /health
        FullyQualifiedDomainName: !Ref SecondaryEndpoint
        Port: 443
        RequestInterval: 30
        FailureThreshold: 3
        MeasureLatency: true
        EnableSNI: true

      HealthCheckTags:
        - Key: Name
          Value: secondary-endpoint-health
        - Key: Region
          Value: eu-west-1

  # Hosted Zone (assume already exists)
  # We'll reference it by name

  # Primary record set (failover primary)
  PrimaryRecordSet:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneName: !Sub '${DomainName}.'
      Name: !Ref DomainName
      Type: A
      SetIdentifier: primary-eu-south-1
      Failover: PRIMARY
      HealthCheckId: !Ref PrimaryHealthCheck
      AliasTarget:
        HostedZoneId: Z1234567890ABC  # ALB hosted zone ID for eu-south-1
        DNSName: !Ref PrimaryEndpoint
        EvaluateTargetHealth: true

  # Secondary record set (failover secondary)
  SecondaryRecordSet:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneName: !Sub '${DomainName}.'
      Name: !Ref DomainName
      Type: A
      SetIdentifier: secondary-eu-west-1
      Failover: SECONDARY
      HealthCheckId: !Ref SecondaryHealthCheck
      AliasTarget:
        HostedZoneId: Z9876543210XYZ  # ALB hosted zone ID for eu-west-1
        DNSName: !Ref SecondaryEndpoint
        EvaluateTargetHealth: true

  # CloudWatch alarms for health check failures
  PrimaryHealthCheckAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: route53-primary-endpoint-unhealthy
      AlarmDescription: Alert when primary endpoint fails health checks
      MetricName: HealthCheckStatus
      Namespace: AWS/Route53
      Statistic: Minimum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: LessThanThreshold
      Dimensions:
        - Name: HealthCheckId
          Value: !Ref PrimaryHealthCheck
      TreatMissingData: breaching

  SecondaryHealthCheckAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: route53-secondary-endpoint-unhealthy
      AlarmDescription: Alert when secondary endpoint fails health checks
      MetricName: HealthCheckStatus
      Namespace: AWS/Route53
      Statistic: Minimum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: LessThanThreshold
      Dimensions:
        - Name: HealthCheckId
          Value: !Ref SecondaryHealthCheck
      TreatMissingData: breaching

Outputs:
  PrimaryHealthCheckId:
    Description: Primary health check ID
    Value: !Ref PrimaryHealthCheck

  SecondaryHealthCheckId:
    Description: Secondary health check ID
    Value: !Ref SecondaryHealthCheck

  DomainName:
    Description: Configured domain name
    Value: !Ref DomainName
```

#### Python Script - Test Route 53 Failover

```python
#!/usr/bin/env python3
"""
test_route53_failover.py - Test Route 53 DNS failover

Simulates primary region failure and measures DNS failover time.
"""

import boto3
import requests
import time
import socket
from datetime import datetime
from typing import Dict, List

# Configuration
DOMAIN_NAME = "api.ai-support.example.com"
PRIMARY_REGION = "eu-south-1"
SECONDARY_REGION = "eu-west-1"
PRIMARY_HEALTH_CHECK_ID = "abc123-primary"
SECONDARY_HEALTH_CHECK_ID = "xyz789-secondary"

# AWS clients
route53 = boto3.client('route53')
cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')  # Route53 metrics in us-east-1


def get_health_check_status(health_check_id: str) -> Dict:
    """Get current status of a health check."""
    response = route53.get_health_check_status(
        HealthCheckId=health_check_id
    )

    checkers = response.get('HealthCheckObservations', [])

    if not checkers:
        return {'status': 'UNKNOWN', 'healthy_count': 0, 'total_count': 0}

    healthy = sum(1 for checker in checkers if checker.get('StatusReport', {}).get('Status') == 'Success')
    total = len(checkers)

    # Health check is considered healthy if majority of checkers report success
    overall_healthy = healthy > (total / 2)

    return {
        'status': 'HEALTHY' if overall_healthy else 'UNHEALTHY',
        'healthy_count': healthy,
        'total_count': total,
        'checkers': checkers
    }


def resolve_domain(domain: str) -> List[str]:
    """Resolve domain to IP addresses."""
    try:
        return socket.gethostbyname_ex(domain)[2]
    except socket.gaierror as e:
        print(f"  ✗ DNS resolution failed: {e}")
        return []


def test_endpoint(url: str) -> Dict:
    """Test endpoint availability."""
    try:
        start_time = time.time()
        response = requests.get(url, timeout=5, verify=True)
        latency = time.time() - start_time

        return {
            'success': True,
            'status_code': response.status_code,
            'latency': latency,
            'error': None
        }
    except Exception as e:
        return {
            'success': False,
            'status_code': None,
            'latency': None,
            'error': str(e)
        }


def simulate_primary_failure():
    """
    Simulate primary endpoint failure.

    In real scenario, this would be actual infrastructure failure.
    For testing, we can:
    1. Stop the ALB target group (requires permissions)
    2. Block health check path in WAF
    3. Modify security group to block Route53 health checkers

    This example shows what would happen, but doesn't actually cause failure.
    """
    print("\n⚠️  NOTE: In production, primary failure would be triggered by:")
    print("  - Infrastructure failure (EC2/ALB down)")
    print("  - Regional outage")
    print("  - Manual intervention (disable target group)")
    print("\n  For testing, manually disable primary endpoint or wait for natural failure.")


def monitor_failover(duration: int = 300):
    """
    Monitor DNS resolution and endpoint availability during failover.

    Args:
        duration: How long to monitor (seconds)
    """
    print(f"\n🔍 Monitoring failover for {duration}s...")
    print("=" * 60)

    start_time = time.time()
    results = []

    while time.time() - start_time < duration:
        elapsed = time.time() - start_time

        # Check health check statuses
        primary_health = get_health_check_status(PRIMARY_HEALTH_CHECK_ID)
        secondary_health = get_health_check_status(SECONDARY_HEALTH_CHECK_ID)

        # Resolve DNS
        ips = resolve_domain(DOMAIN_NAME)

        # Test endpoint
        endpoint_result = test_endpoint(f"https://{DOMAIN_NAME}/health")

        # Record result
        result = {
            'timestamp': datetime.utcnow().isoformat(),
            'elapsed': elapsed,
            'primary_health': primary_health['status'],
            'secondary_health': secondary_health['status'],
            'resolved_ips': ips,
            'endpoint_status': endpoint_result['status_code'],
            'endpoint_success': endpoint_result['success'],
            'latency': endpoint_result.get('latency')
        }

        results.append(result)

        # Print status
        print(f"\n[{elapsed:.0f}s] Status:")
        print(f"  Primary Health: {primary_health['status']} "
              f"({primary_health['healthy_count']}/{primary_health['total_count']} checkers)")
        print(f"  Secondary Health: {secondary_health['status']} "
              f"({secondary_health['healthy_count']}/{secondary_health['total_count']} checkers)")
        print(f"  Resolved IPs: {', '.join(ips) if ips else 'None'}")
        print(f"  Endpoint: {'✓' if endpoint_result['success'] else '✗'} "
              f"({endpoint_result.get('status_code', 'N/A')})")

        if endpoint_result.get('latency'):
            print(f"  Latency: {endpoint_result['latency']*1000:.0f}ms")

        time.sleep(10)  # Check every 10 seconds

    return results


def analyze_results(results: List[Dict]):
    """Analyze monitoring results and calculate failover metrics."""
    print("\n📊 Failover Analysis")
    print("=" * 60)

    # Detect failover event
    failover_detected = False
    failover_time = None
    recovery_time = None

    for i in range(1, len(results)):
        prev = results[i-1]
        curr = results[i]

        # Detect when primary became unhealthy
        if prev['primary_health'] == 'HEALTHY' and curr['primary_health'] == 'UNHEALTHY':
            failover_time = curr['elapsed']
            failover_detected = True
            print(f"\n⚠️  Primary failure detected at {failover_time:.0f}s")

        # Detect when endpoint recovered (via secondary)
        if failover_detected and not recovery_time:
            if not prev['endpoint_success'] and curr['endpoint_success']:
                recovery_time = curr['elapsed']
                rto = recovery_time - failover_time
                print(f"✅ Service recovered at {recovery_time:.0f}s")
                print(f"   RTO: {rto:.1f}s ({rto/60:.2f} minutes)")

    # Calculate availability
    total_checks = len(results)
    successful_checks = sum(1 for r in results if r['endpoint_success'])
    availability = (successful_checks / total_checks) * 100

    print(f"\n📈 Metrics:")
    print(f"  Total Checks: {total_checks}")
    print(f"  Successful: {successful_checks}")
    print(f"  Availability: {availability:.2f}%")

    if failover_detected:
        downtime = recovery_time - failover_time if recovery_time else None
        if downtime:
            print(f"  Downtime: {downtime:.1f}s ({downtime/60:.2f} minutes)")
            print(f"  Impact: {(1 - availability/100) * 100:.2f}% requests failed")

    # DNS TTL impact
    print(f"\n🌐 DNS Considerations:")
    print(f"  Route 53 TTL: 60s (default)")
    print(f"  Client DNS cache: Varies (60-300s typical)")
    print(f"  Total DNS failover time: 90-360s (TTL + health check interval)")


def main():
    """Execute Route 53 failover test."""
    print("=" * 60)
    print("Route 53 DNS Failover Test")
    print("=" * 60)
    print(f"Domain: {DOMAIN_NAME}")

    # Pre-test status
    print("\n1️⃣  Pre-Test Status")
    print("-" * 60)

    primary_health = get_health_check_status(PRIMARY_HEALTH_CHECK_ID)
    secondary_health = get_health_check_status(SECONDARY_HEALTH_CHECK_ID)

    print(f"Primary Region ({PRIMARY_REGION}):")
    print(f"  Health Check: {primary_health['status']}")
    print(f"  Checkers: {primary_health['healthy_count']}/{primary_health['total_count']} healthy")

    print(f"\nSecondary Region ({SECONDARY_REGION}):")
    print(f"  Health Check: {secondary_health['status']}")
    print(f"  Checkers: {secondary_health['healthy_count']}/{secondary_health['total_count']} healthy")

    # Current DNS resolution
    print(f"\nCurrent DNS Resolution:")
    ips = resolve_domain(DOMAIN_NAME)
    for ip in ips:
        print(f"  {ip}")

    # Endpoint test
    print(f"\nEndpoint Test:")
    result = test_endpoint(f"https://{DOMAIN_NAME}/health")
    print(f"  Status: {'✓ Available' if result['success'] else '✗ Unavailable'}")
    if result['success']:
        print(f"  Status Code: {result['status_code']}")
        print(f"  Latency: {result['latency']*1000:.0f}ms")

    # Instructions for failover test
    print("\n2️⃣  Failover Test")
    print("-" * 60)
    simulate_primary_failure()

    input("\nPress Enter when ready to start monitoring (after simulating failure)...")

    # Monitor failover
    results = monitor_failover(duration=300)

    # Analyze results
    analyze_results(results)

    print("\n" + "=" * 60)
    print("Test Complete")
    print("=" * 60)


if __name__ == '__main__':
    main()
```

### Esempio 6: Lambda Multi-Region Deployment

Implementiamo deployment cross-region di Lambda functions per ridondanza.

#### Python Script - Deploy Lambda to Multiple Regions

```python
#!/usr/bin/env python3
"""
deploy_lambda_multi_region.py - Deploy Lambda function to multiple regions

Automates cross-region deployment for disaster recovery.
"""

import boto3
import zipfile
import io
import hashlib
import json
from typing import Dict, List

# Configuration
FUNCTION_NAME = "ticket-classifier"
REGIONS = ["eu-south-1", "eu-west-1"]
RUNTIME = "python3.11"
HANDLER = "index.handler"
MEMORY_SIZE = 512
TIMEOUT = 30

# Lambda function code (example)
LAMBDA_CODE = """
import json
import boto3
import os

def handler(event, context):
    \"\"\"Ticket classification Lambda function.\"\"\"

    # Get region from environment
    region = os.environ.get('AWS_REGION', 'unknown')

    # Simulate classification
    ticket_text = event.get('ticket_text', '')

    result = {
        'ticket_id': event.get('ticket_id'),
        'classification': 'technical_issue',
        'confidence': 0.95,
        'processed_in_region': region,
        'timestamp': context.request_id
    }

    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }
"""


def create_deployment_package() -> bytes:
    """Create Lambda deployment package (ZIP)."""
    zip_buffer = io.BytesIO()

    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zip_file:
        # Add Lambda code
        zip_file.writestr('index.py', LAMBDA_CODE)

        # In production, add dependencies from requirements.txt
        # For this example, we keep it simple

    return zip_buffer.getvalue()


def calculate_code_hash(code: bytes) -> str:
    """Calculate SHA256 hash of deployment package."""
    return hashlib.sha256(code).hexdigest()


def create_or_update_function(region: str, code: bytes, role_arn: str) -> Dict:
    """Create or update Lambda function in specified region."""
    lambda_client = boto3.client('lambda', region_name=region)

    try:
        # Try to get existing function
        response = lambda_client.get_function(FunctionName=FUNCTION_NAME)
        existing = True
        print(f"  Found existing function in {region}")

    except lambda_client.exceptions.ResourceNotFoundException:
        existing = False
        print(f"  Function not found in {region}, will create new")

    if existing:
        # Update function code
        print(f"  Updating function code...")
        response = lambda_client.update_function_code(
            FunctionName=FUNCTION_NAME,
            ZipFile=code,
            Publish=True  # Publish new version
        )

        # Update configuration
        print(f"  Updating function configuration...")
        config_response = lambda_client.update_function_configuration(
            FunctionName=FUNCTION_NAME,
            Runtime=RUNTIME,
            Handler=HANDLER,
            MemorySize=MEMORY_SIZE,
            Timeout=TIMEOUT,
            Environment={
                'Variables': {
                    'DEPLOYMENT_REGION': region,
                    'DEPLOYMENT_TIME': str(int(time.time()))
                }
            }
        )

        version = response['Version']
        print(f"  ✓ Updated to version {version}")

    else:
        # Create new function
        print(f"  Creating new function...")
        response = lambda_client.create_function(
            FunctionName=FUNCTION_NAME,
            Runtime=RUNTIME,
            Role=role_arn,
            Handler=HANDLER,
            Code={'ZipFile': code},
            MemorySize=MEMORY_SIZE,
            Timeout=TIMEOUT,
            Publish=True,
            Environment={
                'Variables': {
                    'DEPLOYMENT_REGION': region,
                    'DEPLOYMENT_TIME': str(int(time.time()))
                }
            },
            Tags={
                'Environment': 'production',
                'DeployedBy': 'multi-region-script'
            }
        )

        version = response['Version']
        print(f"  ✓ Created version {version}")

    return {
        'region': region,
        'function_arn': response['FunctionArn'],
        'version': version,
        'code_sha256': response['CodeSha256']
    }


def create_alias(region: str, version: str, alias_name: str = 'production'):
    """Create or update alias pointing to specific version."""
    lambda_client = boto3.client('lambda', region_name=region)

    try:
        # Try to update existing alias
        response = lambda_client.update_alias(
            FunctionName=FUNCTION_NAME,
            Name=alias_name,
            FunctionVersion=version
        )
        print(f"  ✓ Updated alias '{alias_name}' -> version {version}")

    except lambda_client.exceptions.ResourceNotFoundException:
        # Create new alias
        response = lambda_client.create_alias(
            FunctionName=FUNCTION_NAME,
            Name=alias_name,
            FunctionVersion=version,
            Description=f'Production alias for {FUNCTION_NAME}'
        )
        print(f"  ✓ Created alias '{alias_name}' -> version {version}")

    return response['AliasArn']


def test_function(region: str, alias: str = 'production'):
    """Test Lambda function in specific region."""
    lambda_client = boto3.client('lambda', region_name=region)

    test_event = {
        'ticket_id': 'TEST-001',
        'ticket_text': 'Server not responding to ping'
    }

    try:
        response = lambda_client.invoke(
            FunctionName=f"{FUNCTION_NAME}:{alias}",
            InvocationType='RequestResponse',
            Payload=json.dumps(test_event)
        )

        payload = json.loads(response['Payload'].read())

        print(f"  ✓ Test successful")
        print(f"    Response: {payload}")

        return True

    except Exception as e:
        print(f"  ✗ Test failed: {str(e)}")
        return False


def get_function_concurrency(region: str) -> Dict:
    """Get function concurrency configuration."""
    lambda_client = boto3.client('lambda', region_name=region)

    try:
        response = lambda_client.get_function_concurrency(
            FunctionName=FUNCTION_NAME
        )
        return response
    except:
        return None


def set_reserved_concurrency(region: str, concurrency: int):
    """Set reserved concurrency for function."""
    lambda_client = boto3.client('lambda', region_name=region)

    response = lambda_client.put_function_concurrency(
        FunctionName=FUNCTION_NAME,
        ReservedConcurrentExecutions=concurrency
    )

    print(f"  ✓ Set reserved concurrency: {concurrency}")
    return response


import time

def main():
    """Execute multi-region deployment."""
    print("=" * 60)
    print("Lambda Multi-Region Deployment")
    print("=" * 60)
    print(f"Function: {FUNCTION_NAME}")
    print(f"Regions: {', '.join(REGIONS)}")

    # Create deployment package
    print("\n1️⃣  Creating Deployment Package")
    print("-" * 60)
    code = create_deployment_package()
    code_hash = calculate_code_hash(code)
    print(f"  Package size: {len(code)} bytes")
    print(f"  SHA256: {code_hash[:16]}...")

    # Get IAM role (assume exists)
    iam = boto3.client('iam')
    role_name = 'lambda-execution-role'

    try:
        role_response = iam.get_role(RoleName=role_name)
        role_arn = role_response['Role']['Arn']
        print(f"  Using IAM role: {role_arn}")
    except:
        print(f"  ✗ IAM role '{role_name}' not found")
        print("  Create role first with appropriate permissions")
        return

    # Deploy to each region
    print("\n2️⃣  Deploying to Regions")
    print("-" * 60)

    deployments = []

    for region in REGIONS:
        print(f"\n🌍 Region: {region}")

        # Deploy function
        deployment = create_or_update_function(region, code, role_arn)
        deployments.append(deployment)

        # Create alias
        alias_arn = create_alias(region, deployment['version'])

        # Set reserved concurrency (production best practice)
        set_reserved_concurrency(region, 100)

        # Wait for function to be ready
        time.sleep(2)

        # Test function
        print(f"  Testing function...")
        test_function(region, 'production')

    # Verify deployment consistency
    print("\n3️⃣  Verification")
    print("-" * 60)

    code_hashes = [d['code_sha256'] for d in deployments]
    all_match = len(set(code_hashes)) == 1

    print(f"  Code SHA256 consistency: {'✓ All match' if all_match else '✗ Mismatch detected'}")

    for deployment in deployments:
        print(f"\n  {deployment['region']}:")
        print(f"    ARN: {deployment['function_arn']}")
        print(f"    Version: {deployment['version']}")
        print(f"    Code SHA256: {deployment['code_sha256'][:16]}...")

    # Summary
    print("\n" + "=" * 60)
    print("DEPLOYMENT SUMMARY")
    print("=" * 60)
    print(f"✓ Deployed to {len(deployments)} regions")
    print(f"✓ All regions have identical code: {all_match}")
    print(f"✓ Production alias configured")
    print(f"✓ Reserved concurrency: 100 per region")
    print("\n🎯 Functions are ready for cross-region traffic")
    print("=" * 60)


if __name__ == '__main__':
    main()
```

### Esempio 7: AWS Backup - Automated Backup Setup

Configuriamo AWS Backup per centralizzare i backup cross-region.

#### CloudFormation - AWS Backup Configuration

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'AWS Backup configuration for automated cross-region backups'

Parameters:
  EnvironmentName:
    Type: String
    Default: production

Resources:
  # Backup Vault (primary region)
  PrimaryBackupVault:
    Type: AWS::Backup::BackupVault
    Properties:
      BackupVaultName: !Sub '${EnvironmentName}-primary-vault'
      EncryptionKeyArn: !GetAtt BackupVaultKey.Arn
      Notifications:
        BackupVaultEvents:
          - BACKUP_JOB_STARTED
          - BACKUP_JOB_COMPLETED
          - BACKUP_JOB_FAILED
          - RESTORE_JOB_STARTED
          - RESTORE_JOB_COMPLETED
          - RESTORE_JOB_FAILED
        SNSTopicArn: !Ref BackupNotificationTopic

  # KMS key for backup vault encryption
  BackupVaultKey:
    Type: AWS::KMS::Key
    Properties:
      Description: KMS key for AWS Backup vault encryption
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'

          - Sid: Allow AWS Backup to use the key
            Effect: Allow
            Principal:
              Service: backup.amazonaws.com
            Action:
              - 'kms:CreateGrant'
              - 'kms:GenerateDataKey'
              - 'kms:Decrypt'
              - 'kms:DescribeKey'
            Resource: '*'

  BackupVaultKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: !Sub 'alias/${EnvironmentName}-backup-key'
      TargetKeyId: !Ref BackupVaultKey

  # Backup Plan
  BackupPlan:
    Type: AWS::Backup::BackupPlan
    Properties:
      BackupPlan:
        BackupPlanName: !Sub '${EnvironmentName}-backup-plan'

        # Backup rules
        BackupPlanRule:
          # Daily backups
          - RuleName: DailyBackups
            TargetBackupVault: !Ref PrimaryBackupVault
            ScheduleExpression: 'cron(0 3 * * ? *)'  # 3 AM UTC daily
            StartWindowMinutes: 60
            CompletionWindowMinutes: 120
            Lifecycle:
              DeleteAfterDays: 7  # Retain for 7 days
              MoveToColdStorageAfterDays: 1  # Move to cold storage after 1 day
            RecoveryPointTags:
              BackupType: Daily
              Environment: !Ref EnvironmentName

            # Copy to secondary region
            CopyActions:
              - DestinationBackupVaultArn: !Sub 'arn:aws:backup:eu-west-1:${AWS::AccountId}:backup-vault:${EnvironmentName}-replica-vault'
                Lifecycle:
                  DeleteAfterDays: 7

          # Weekly backups (longer retention)
          - RuleName: WeeklyBackups
            TargetBackupVault: !Ref PrimaryBackupVault
            ScheduleExpression: 'cron(0 4 ? * SUN *)'  # 4 AM UTC every Sunday
            StartWindowMinutes: 60
            CompletionWindowMinutes: 180
            Lifecycle:
              DeleteAfterDays: 35  # Retain for 5 weeks
              MoveToColdStorageAfterDays: 7
            RecoveryPointTags:
              BackupType: Weekly
              Environment: !Ref EnvironmentName

            CopyActions:
              - DestinationBackupVaultArn: !Sub 'arn:aws:backup:eu-west-1:${AWS::AccountId}:backup-vault:${EnvironmentName}-replica-vault'
                Lifecycle:
                  DeleteAfterDays: 35

          # Monthly backups (long-term retention)
          - RuleName: MonthlyBackups
            TargetBackupVault: !Ref PrimaryBackupVault
            ScheduleExpression: 'cron(0 5 1 * ? *)'  # 5 AM UTC first day of month
            StartWindowMinutes: 120
            CompletionWindowMinutes: 360
            Lifecycle:
              DeleteAfterDays: 365  # Retain for 1 year
              MoveToColdStorageAfterDays: 30
            RecoveryPointTags:
              BackupType: Monthly
              Environment: !Ref EnvironmentName

            CopyActions:
              - DestinationBackupVaultArn: !Sub 'arn:aws:backup:eu-west-1:${AWS::AccountId}:backup-vault:${EnvironmentName}-replica-vault'
                Lifecycle:
                  DeleteAfterDays: 365

      BackupPlanTags:
        Environment: !Ref EnvironmentName

  # Backup Selection (what to backup)
  BackupSelection:
    Type: AWS::Backup::BackupSelection
    Properties:
      BackupPlanId: !Ref BackupPlan
      BackupSelection:
        SelectionName: !Sub '${EnvironmentName}-resources'
        IamRoleArn: !GetAtt BackupRole.Arn

        # Select resources by tags
        ListOfTags:
          - ConditionType: STRINGEQUALS
            ConditionKey: Backup
            ConditionValue: 'true'

        # Explicitly include specific resources
        Resources:
          - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${EnvironmentName}-tickets'
          - !Sub 'arn:aws:rds:${AWS::Region}:${AWS::AccountId}:db:${EnvironmentName}-analytics-db'

  # IAM Role for AWS Backup
  BackupRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: backup.amazonaws.com
            Action: 'sts:AssumeRole'
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup'
        - 'arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores'
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-backup-role'

  # SNS Topic for backup notifications
  BackupNotificationTopic:
    Type: AWS::SNS::Topic
    Properties:
      DisplayName: AWS Backup Notifications
      Subscription:
        - Endpoint: ops-team@example.com
          Protocol: email

  # CloudWatch Alarms
  BackupJobFailedAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-backup-job-failed'
      AlarmDescription: Alert when backup job fails
      MetricName: NumberOfBackupJobsFailed
      Namespace: AWS/Backup
      Statistic: Sum
      Period: 3600
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      TreatMissingData: notBreaching

Outputs:
  BackupVaultArn:
    Description: Backup vault ARN
    Value: !GetAtt PrimaryBackupVault.BackupVaultArn

  BackupPlanId:
    Description: Backup plan ID
    Value: !Ref BackupPlan

  BackupPlanArn:
    Description: Backup plan ARN
    Value: !GetAtt BackupPlan.BackupPlanArn
```

### Esempio 8: DR Runbook - Step-by-Step Recovery

Creiamo un runbook dettagliato per il disaster recovery.

#### DR Runbook Markdown

```markdown
# Disaster Recovery Runbook - AI Technical Support System

**Version**: 1.0
**Last Updated**: 2025-11-18
**Owner**: Platform Engineering Team
**Review Frequency**: Quarterly

## Emergency Contacts

| Role | Name | Phone | Email |
|------|------|-------|-------|
| Incident Commander | John Doe | +39-xxx-xxxx | john.doe@example.com |
| Technical Lead | Jane Smith | +39-xxx-xxxx | jane.smith@example.com |
| AWS Support | - | - | aws-support@example.com |
| On-Call Engineer | Rotation | +39-xxx-xxxx | oncall@example.com |

## Severity Levels

### SEV-1: Complete System Outage
- **Impact**: All customers unable to use the system
- **RTO**: 4 hours
- **RPO**: 1 hour
- **Response Time**: Immediate
- **Escalation**: Incident Commander + CTO

### SEV-2: Partial Outage
- **Impact**: Degraded service, some features unavailable
- **RTO**: 12 hours
- **RPO**: 4 hours
- **Response Time**: 15 minutes
- **Escalation**: Incident Commander

### SEV-3: Performance Degradation
- **Impact**: Slow response times, no data loss
- **RTO**: 24 hours
- **RPO**: N/A
- **Response Time**: 1 hour
- **Escalation**: On-Call Engineer

## Disaster Scenarios

### Scenario 1: Primary Region (eu-south-1) Complete Failure

#### Detection
```bash
# Automated CloudWatch Alarm triggers
# Manual verification
aws cloudwatch describe-alarm-history \
  --alarm-name "primary-region-health" \
  --max-records 10
```

#### Decision Tree
```
Is primary region completely unavailable?
├─ YES → Execute full failover to eu-west-1
└─ NO  → Is it partial failure?
    ├─ YES → Execute partial failover (affected services only)
    └─ NO  → False alarm, investigate and resolve
```

#### Failover Steps

**Step 1: Declare Incident (0-5 minutes)**

```bash
# 1.1 Confirm outage
curl -I https://api.ai-support.example.com/health
# Expected: Timeout or 503

# 1.2 Check AWS Service Health
aws health describe-events \
  --filter eventTypeCategories=issue \
  --region eu-south-1

# 1.3 Notify stakeholders
python notify_incident.py --severity SEV-1 --type region-failure
```

**Step 2: Activate DR Mode (5-15 minutes)**

```bash
# 2.1 Update Route 53 health check (force failover)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch file://failover-to-secondary.json

# failover-to-secondary.json
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "api.ai-support.example.com",
      "Type": "A",
      "SetIdentifier": "primary-eu-south-1",
      "Failover": "PRIMARY",
      "HealthCheckId": "healthcheck-id",
      "AliasTarget": {
        "HostedZoneId": "Z...",
        "DNSName": "secondary-alb.eu-west-1.elb.amazonaws.com",
        "EvaluateTargetHealth": false
      }
    }
  }]
}

# 2.2 Verify DNS propagation
dig api.ai-support.example.com +short
# Should return eu-west-1 IPs

# 2.3 Scale up secondary region capacity
aws lambda put-function-concurrency \
  --function-name ticket-processor \
  --reserved-concurrent-executions 500 \
  --region eu-west-1

aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:ticket-processor:production \
  --scalable-dimension lambda:function:ProvisionedConcurrentExecutions \
  --min-capacity 50 \
  --max-capacity 200 \
  --region eu-west-1
```

**Step 3: Verify Secondary Region (15-30 minutes)**

```bash
# 3.1 Test critical endpoints
bash test_endpoints.sh eu-west-1

# 3.2 Verify data replication
python verify_data_replication.py \
  --source eu-south-1 \
  --target eu-west-1

# 3.3 Check DynamoDB Global Tables
aws dynamodb describe-table \
  --table-name production-tickets \
  --region eu-west-1 \
  --query 'Table.Replicas[?RegionName==`eu-west-1`].ReplicaStatus'

# 3.4 Monitor error rates
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApiGateway \
  --metric-name 5XXError \
  --dimensions Name=ApiName,Value=ai-support-api \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --region eu-west-1
```

**Step 4: Communicate Status (Ongoing)**

```bash
# 4.1 Update status page
curl -X POST https://status.ai-support.example.com/api/incidents \
  -H "Authorization: Bearer $API_KEY" \
  -d '{
    "name": "Primary Region Outage",
    "status": "investigating",
    "message": "We are experiencing issues with our primary datacenter. Traffic has been redirected to our secondary location.",
    "components": ["api", "dashboard"]
  }'

# 4.2 Send customer notifications
python send_customer_notification.py \
  --template dr-failover \
  --severity high
```

**Step 5: Monitor and Stabilize (30-60 minutes)**

```bash
# 5.1 Real-time monitoring dashboard
aws cloudwatch get-dashboard \
  --dashboard-name production-overview-eu-west-1

# 5.2 Check for cascading failures
python check_service_health.py --region eu-west-1 --verbose

# 5.3 Verify all integrations
bash test_integrations.sh
# - SageMaker endpoints
# - Bedrock availability
# - OpenSearch cluster health
# - External APIs
```

#### Rollback/Failback Procedure

**When to Failback**:
- Primary region is fully restored and stable for 2+ hours
- All data is synchronized
- Approval from Incident Commander

**Failback Steps**:

```bash
# 1. Verify primary region health
python verify_region_health.py --region eu-south-1 --thorough

# 2. Sync data from secondary to primary (if needed)
python sync_dynamodb_tables.py \
  --source eu-west-1 \
  --target eu-south-1 \
  --dry-run

# Review sync plan
# Execute if OK
python sync_dynamodb_tables.py \
  --source eu-west-1 \
  --target eu-south-1 \
  --execute

# 3. Gradual traffic shift (10% → 50% → 100%)
python shift_traffic.py \
  --from eu-west-1 \
  --to eu-south-1 \
  --percentage 10 \
  --monitor-duration 15

# Monitor for 15 minutes, then continue
python shift_traffic.py --percentage 50 --monitor-duration 30
python shift_traffic.py --percentage 100

# 4. Scale down secondary region
aws lambda put-function-concurrency \
  --function-name ticket-processor \
  --reserved-concurrent-executions 100 \
  --region eu-west-1

# 5. Update status
curl -X PATCH https://status.ai-support.example.com/api/incidents/latest \
  -d '{"status": "resolved", "message": "All systems operational"}'
```

### Scenario 2: RDS Database Failure

#### Detection
```bash
# Check RDS status
aws rds describe-db-instances \
  --db-instance-identifier production-analytics-db \
  --query 'DBInstances[0].DBInstanceStatus'
```

#### Recovery Steps

```bash
# 1. If Multi-AZ failover hasn't triggered automatically
aws rds reboot-db-instance \
  --db-instance-identifier production-analytics-db \
  --force-failover

# 2. Monitor failover
aws rds describe-events \
  --source-identifier production-analytics-db \
  --duration 60

# 3. Verify new endpoint
aws rds describe-db-instances \
  --db-instance-identifier production-analytics-db \
  --query 'DBInstances[0].Endpoint'

# 4. Test connectivity
psql -h <new-endpoint> -U postgres -d analytics -c "SELECT 1"

# 5. If complete failure, restore from snapshot
SNAPSHOT_ID=$(aws rds describe-db-snapshots \
  --db-instance-identifier production-analytics-db \
  --query 'reverse(sort_by(DBSnapshots, &SnapshotCreateTime))[0].DBSnapshotIdentifier' \
  --output text)

aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier production-analytics-db-restore \
  --db-snapshot-identifier $SNAPSHOT_ID \
  --db-instance-class db.t3.medium \
  --multi-az
```

### Scenario 3: DynamoDB Table Corruption

#### Recovery Steps

```bash
# 1. Stop writes to table
python emergency_stop_writes.py --table production-tickets

# 2. Create on-demand backup
aws dynamodb create-backup \
  --table-name production-tickets \
  --backup-name emergency-backup-$(date +%Y%m%d-%H%M%S)

# 3. Point-in-time restore
TARGET_TIME=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S)

aws dynamodb restore-table-to-point-in-time \
  --source-table-name production-tickets \
  --target-table-name production-tickets-restored \
  --restore-date-time $TARGET_TIME

# 4. Verify restored data
python verify_table_integrity.py --table production-tickets-restored

# 5. Switch application to restored table
python switch_table.py \
  --from production-tickets \
  --to production-tickets-restored

# 6. Monitor for issues
python monitor_table_health.py --duration 3600
```

## Post-Incident Review

### Required Within 48 Hours

1. **Incident Timeline**: Document all events with timestamps
2. **Root Cause Analysis**: Identify what caused the failure
3. **Impact Assessment**: Quantify customer impact, downtime, data loss
4. **Lessons Learned**: What went well, what didn't
5. **Action Items**: Preventive measures to avoid recurrence

### Template

```markdown
## Incident Post-Mortem: [Incident Name]

**Date**: YYYY-MM-DD
**Duration**: X hours Y minutes
**Severity**: SEV-X
**Services Affected**: [List]

### Timeline
| Time | Event |
|------|-------|
| 14:00 | Initial alert triggered |
| 14:05 | Incident confirmed |
| 14:15 | Failover initiated |
| ... | ... |

### Root Cause
[Detailed explanation]

### Impact
- Customers affected: X
- Requests failed: Y
- Revenue impact: $Z
- Data loss: None / X records

### What Went Well
- [Item 1]
- [Item 2]

### What Went Wrong
- [Item 1]
- [Item 2]

### Action Items
| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| [Action 1] | [Name] | YYYY-MM-DD | High |
```

## Testing Schedule

| Test Type | Frequency | Last Executed | Next Due |
|-----------|-----------|---------------|----------|
| RDS Failover | Monthly | 2025-10-15 | 2025-11-15 |
| Route 53 Failover | Quarterly | 2025-09-01 | 2025-12-01 |
| Full DR Drill | Annually | 2025-01-15 | 2026-01-15 |
| Backup Restore | Monthly | 2025-10-20 | 2025-11-20 |
| Runbook Review | Quarterly | 2025-10-01 | 2026-01-01 |

## Appendix

### Useful Commands

```bash
# Check overall system health
python check_system_health.py --comprehensive

# Get current traffic distribution
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApiGateway \
  --metric-name Count \
  --dimensions Name=ApiName,Value=ai-support-api \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics Sum \
  --region eu-south-1

# List all recent backups
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name production-primary-vault \
  --max-results 10

# Test cross-region connectivity
python test_cross_region.py --source eu-south-1 --target eu-west-1
```

### Configuration Files

**Route 53 Failover Configuration**: `/ops/route53/failover-config.json`
**Lambda Deployment**: `/ops/lambda/multi-region-deploy.sh`
**Backup Policies**: `/ops/backup/backup-plan.yaml`
**Monitoring Dashboards**: `/ops/cloudwatch/dashboards/`
```

## Best Practices

### Design Principles

#### 1. Assume Everything Fails

**Mindset**: Don't ask "Will it fail?", ask "When will it fail and how do we recover?"

```python
# Bad: Single point of failure
def process_ticket(ticket_id):
    result = bedrock_client.invoke_model(...)  # What if Bedrock is down?
    return result

# Good: Fallback strategy
def process_ticket(ticket_id):
    try:
        result = bedrock_client.invoke_model(...)
        return result
    except BedrockThrottlingException:
        # Fallback to secondary model
        return backup_model_client.invoke(...)
    except BedrockUnavailableException:
        # Queue for retry
        sqs.send_message(queue_url, ticket_id)
        return {'status': 'queued'}
```

#### 2. Build Redundancy at Every Layer

**Multi-AZ**: For all stateful services
**Multi-Region**: For critical data
**Multi-Model**: For AI/ML workloads

| Layer | Redundancy Strategy |
|-------|---------------------|
| DNS | Route 53 with health checks across regions |
| Load Balancer | ALB nodes in 3 AZs |
| Compute | Lambda auto-distribution, ECS tasks in multiple AZs |
| Database | RDS Multi-AZ, DynamoDB Global Tables |
| Storage | S3 (11 9s durability), cross-region replication |
| AI Models | Multiple Bedrock models, SageMaker multi-AZ endpoints |

#### 3. Automate Recovery

**Manual intervention should be last resort**

```yaml
# CloudFormation - Auto-healing ECS
Resources:
  ECSService:
    Type: AWS::ECS::Service
    Properties:
      DesiredCount: 3
      HealthCheckGracePeriodSeconds: 60
      DeploymentConfiguration:
        MinimumHealthyPercent: 66  # Allow rolling updates
        MaximumPercent: 200
      # Auto-replace unhealthy tasks
      PlacementStrategies:
        - Type: spread
          Field: attribute:ecs.availability-zone
```

#### 4. Test Failure Scenarios Regularly

**Chaos Engineering**: Deliberately inject failures

```python
# Example: Random AZ failure simulation
import random
from datetime import datetime

def chaos_monkey_az_failure():
    """Simulate AZ failure during testing."""
    azs = ['eu-south-1a', 'eu-south-1b', 'eu-south-1c']
    failed_az = random.choice(azs)

    # Block traffic to this AZ
    ec2 = boto3.client('ec2')

    # Get all subnets in the AZ
    subnets = ec2.describe_subnets(
        Filters=[{'Name': 'availability-zone', 'Values': [failed_az]}]
    )

    for subnet in subnets['Subnets']:
        # Modify NACL to block traffic (reversible)
        print(f"Simulating failure in {failed_az}")
        # Implementation details...

    # Monitor system behavior
    # Verify auto-healing
    # Measure RTO

    # Restore after test
```

#### 5. Monitor Everything

**Observability is not optional**

```python
# Comprehensive health check
def deep_health_check():
    """Multi-layer health verification."""
    health = {
        'timestamp': datetime.utcnow().isoformat(),
        'overall': 'healthy',
        'components': {}
    }

    # Check DynamoDB
    try:
        dynamodb.describe_table(TableName='production-tickets')
        health['components']['dynamodb'] = 'healthy'
    except Exception as e:
        health['components']['dynamodb'] = f'unhealthy: {str(e)}'
        health['overall'] = 'degraded'

    # Check RDS
    try:
        conn = psycopg2.connect(...)
        conn.cursor().execute("SELECT 1")
        health['components']['rds'] = 'healthy'
    except:
        health['components']['rds'] = 'unhealthy'
        health['overall'] = 'degraded'

    # Check OpenSearch
    try:
        opensearch_client.cluster.health()
        health['components']['opensearch'] = 'healthy'
    except:
        health['components']['opensearch'] = 'unhealthy'
        health['overall'] = 'degraded'

    # Check Bedrock availability
    try:
        bedrock_client.list_foundation_models()
        health['components']['bedrock'] = 'healthy'
    except:
        health['components']['bedrock'] = 'unhealthy'
        health['overall'] = 'degraded'

    return health
```

### Operational Excellence

#### 1. Document Everything

- **Runbooks**: Step-by-step procedures for common scenarios
- **Architecture Diagrams**: Always up-to-date
- **Dependency Maps**: Know what depends on what
- **Contact Lists**: Who to call, when

#### 2. Establish Clear RTO/RPO for Each Service

| Service | Tier | RTO | RPO | Strategy |
|---------|------|-----|-----|----------|
| Ticket API | Critical | < 5 min | 0 | Active-Active Multi-Region |
| AI Generation | Critical | < 15 min | 0 | Multi-Model Fallback |
| Knowledge Base | High | < 30 min | < 1 hour | Snapshot + Restore |
| Analytics DB | Medium | < 4 hours | < 4 hours | Multi-AZ + Daily Backup |
| Logs | Low | < 24 hours | < 24 hours | S3 CRR |

#### 3. Implement Circuit Breakers

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=60)
def call_external_api(endpoint):
    """
    Circuit breaker pattern.

    After 5 failures, circuit opens for 60 seconds.
    Prevents cascading failures.
    """
    response = requests.get(endpoint, timeout=5)
    response.raise_for_status()
    return response.json()

# Usage
try:
    data = call_external_api('https://external-service.com/api')
except CircuitBreakerError:
    # Circuit is open, use fallback
    data = get_cached_data()
```

#### 4. Use Bulkheads to Isolate Failures

```python
# Separate Lambda concurrency per tenant to prevent noisy neighbor

tenants = {
    'tenant-a': {
        'function_arn': 'arn:aws:lambda:...:function:processor',
        'reserved_concurrency': 50  # Isolated capacity
    },
    'tenant-b': {
        'function_arn': 'arn:aws:lambda:...:function:processor',
        'reserved_concurrency': 30
    }
}

# If tenant-a has a spike, it won't affect tenant-b
```

#### 5. Implement Graceful Degradation

```python
def generate_response(ticket):
    """Generate ticket response with graceful degradation."""

    # Try primary model (best quality)
    try:
        return bedrock_claude_3_5.invoke(ticket)
    except (Throttling, ServiceUnavailable):
        pass

    # Fallback to faster model (good quality)
    try:
        return bedrock_llama_2.invoke(ticket)
    except Exception:
        pass

    # Last resort: Template-based response (basic quality)
    return template_engine.generate(ticket.category)
```

### Security Best Practices

#### 1. Encrypt Everything

- **At Rest**: KMS encryption for all data stores
- **In Transit**: TLS 1.3 for all communications
- **Backup**: Encrypted backups with separate KMS keys

#### 2. Least Privilege Access

```yaml
# IAM Policy - Minimal DR permissions
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "route53:ChangeResourceRecordSets",  # DNS failover
      "lambda:UpdateFunctionConfiguration",  # Scale functions
      "rds:RebootDBInstance",  # Force RDS failover
      "backup:StartRestoreJob"  # Restore from backup
    ],
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "aws:RequestedRegion": ["eu-south-1", "eu-west-1"]
      }
    }
  }]
}
```

#### 3. Audit Trail

- **CloudTrail**: All API calls logged
- **VPC Flow Logs**: Network traffic monitoring
- **Change Management**: All infrastructure changes via IaC

### Cost Optimization

#### 1. Right-Size DR Capacity

**Don't over-provision standby resources**

```
Production Primary:
- Lambda: 500 concurrent executions
- RDS: db.r5.xlarge
- OpenSearch: 3x r6g.large

DR Standby (Pilot Light):
- Lambda: 100 concurrent (scale on demand)
- RDS: db.t3.medium read replica (promote when needed)
- OpenSearch: 1x t3.medium (restore from snapshot)

Cost Savings: ~70% compared to full hot standby
RTO Impact: +15 minutes (acceptable for our SLA)
```

#### 2. Use S3 Intelligent-Tiering

```yaml
LifecycleConfiguration:
  Rules:
    - Id: AutoArchive
      Status: Enabled
      Transitions:
        - Days: 0
          StorageClass: INTELLIGENT_TIERING
      # S3 automatically moves data to cheapest tier
      # Frequent access: Standard
      # Infrequent access: IA
      # Rarely accessed: Archive Instant Access
```

#### 3. Delete Old Snapshots

```python
def cleanup_old_snapshots(retention_days=30):
    """Delete RDS snapshots older than retention period."""
    rds = boto3.client('rds')

    snapshots = rds.describe_db_snapshots(
        SnapshotType='manual'
    )['DBSnapshots']

    cutoff_date = datetime.now() - timedelta(days=retention_days)

    for snapshot in snapshots:
        if snapshot['SnapshotCreateTime'].replace(tzinfo=None) < cutoff_date:
            print(f"Deleting old snapshot: {snapshot['DBSnapshotIdentifier']}")
            rds.delete_db_snapshot(
                DBSnapshotIdentifier=snapshot['DBSnapshotIdentifier']
            )
```

## Troubleshooting

### Problem 1: RDS Multi-AZ Failover Takes Too Long

**Symptoms**:
- Failover exceeds 5-minute target
- Extended database downtime

**Root Causes**:
1. Large number of connections not properly closed
2. DNS TTL too high
3. Application not handling connection failures

**Solutions**:

```python
# 1. Implement connection pooling with proper timeouts
from sqlalchemy import create_engine, pool

engine = create_engine(
    f'postgresql://{user}:{password}@{endpoint}:5432/{database}',
    poolclass=pool.QueuePool,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,  # Verify connection before using
    pool_recycle=3600,  # Recycle connections after 1 hour
    connect_args={
        'connect_timeout': 5,
        'options': '-c statement_timeout=30000'  # 30s query timeout
    }
)

# 2. Implement retry logic with exponential backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=2, max=60)
)
def execute_query(query):
    with engine.connect() as conn:
        return conn.execute(query)
```

**Prevention**:
- Set appropriate connection timeouts
- Use connection pooling
- Implement health checks
- Monitor failover metrics

### Problem 2: S3 Cross-Region Replication Lag

**Symptoms**:
- Objects not appearing in replica region
- Replication metrics show high latency

**Diagnostics**:

```bash
# Check replication status
aws s3api head-object \
  --bucket primary-bucket \
  --key path/to/object \
  --query 'ReplicationStatus'

# Check replication metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name ReplicationLatency \
  --dimensions Name=SourceBucket,Value=primary-bucket \
              Name=DestinationBucket,Value=replica-bucket \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Maximum,Average
```

**Solutions**:

1. **Check IAM role permissions**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetReplicationConfiguration",
      "s3:ListBucket"
    ],
    "Resource": "arn:aws:s3:::primary-bucket"
  }, {
    "Effect": "Allow",
    "Action": [
      "s3:GetObjectVersionForReplication",
      "s3:GetObjectVersionAcl"
    ],
    "Resource": "arn:aws:s3:::primary-bucket/*"
  }, {
    "Effect": "Allow",
    "Action": [
      "s3:ReplicateObject",
      "s3:ReplicateDelete"
    ],
    "Resource": "arn:aws:s3:::replica-bucket/*"
  }]
}
```

2. **Enable S3 Replication Time Control (RTC)**
```yaml
ReplicationConfiguration:
  Rules:
    - Id: ReplicateWithRTC
      Status: Enabled
      ReplicationTime:
        Status: Enabled
        Time:
          Minutes: 15  # 99.99% of objects within 15 minutes
      Metrics:
        Status: Enabled
```

3. **Check KMS permissions if using encryption**

### Problem 3: DynamoDB Global Tables Conflicts

**Symptoms**:
- Inconsistent data across regions
- Unexpected item versions

**Understanding Conflict Resolution**:

DynamoDB Global Tables use **Last-Writer-Wins** (LWW) based on timestamp.

```python
# Example conflict scenario
# Region 1 writes at T1
table_region1.put_item(Item={
    'id': 'item-1',
    'value': 'A',
    'timestamp': 1637000000000  # T1
})

# Region 2 writes at T2 (later)
table_region2.put_item(Item={
    'id': 'item-1',
    'value': 'B',
    'timestamp': 1637000001000  # T2 (wins)
})

# After convergence, both regions will have value='B'
```

**Solutions**:

1. **Use conditional writes to detect conflicts**
```python
from botocore.exceptions import ClientError

def conditional_update(item_id, new_value, expected_version):
    """Update only if version matches (optimistic locking)."""
    try:
        response = table.update_item(
            Key={'id': item_id},
            UpdateExpression='SET #val = :new_val, #ver = :new_ver',
            ConditionExpression='#ver = :expected_ver',
            ExpressionAttributeNames={
                '#val': 'value',
                '#ver': 'version'
            },
            ExpressionAttributeValues={
                ':new_val': new_value,
                ':expected_ver': expected_version,
                ':new_ver': expected_version + 1
            },
            ReturnValues='ALL_NEW'
        )
        return response['Attributes']

    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            # Conflict detected - item was modified
            print("Conflict detected, retry with latest version")
            raise ConflictError("Item was modified by another writer")
        raise
```

2. **Implement version vectors for complex conflict resolution**

3. **Design for eventual consistency**
```python
# Don't assume immediate consistency
item = table.get_item(Key={'id': 'item-1'})

# This might not reflect recent write in other region yet
# For critical reads, add delays or use reconciliation logic
```

### Problem 4: Lambda Cold Starts During Failover

**Symptoms**:
- High latency spikes when traffic shifts to standby region
- Timeout errors during failover

**Solutions**:

1. **Provisioned Concurrency**
```yaml
LambdaFunction:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: ticket-processor

LambdaAlias:
  Type: AWS::Lambda::Alias
  Properties:
    FunctionName: !Ref LambdaFunction
    Name: production
    FunctionVersion: !GetAtt LambdaVersion.Version
    ProvisionedConcurrencyConfig:
      ProvisionedConcurrentExecutions: 50  # Always warm
```

2. **Warm-up Lambda**
```python
# Scheduled EventBridge rule to keep Lambda warm
def warmup_handler(event, context):
    """Keep Lambda container warm."""
    if event.get('source') == 'warmup':
        return {'status': 'warmed'}

    # Normal processing
    return process_request(event)
```

3. **Application-level caching**
```python
# Cache expensive initializations outside handler
import boto3

# ❌ Bad: Creates new client every invocation
def handler(event, context):
    bedrock = boto3.client('bedrock-runtime')
    result = bedrock.invoke_model(...)

# ✅ Good: Reuses client across invocations
bedrock_client = boto3.client('bedrock-runtime')

def handler(event, context):
    result = bedrock_client.invoke_model(...)
```

### Problem 5: Route 53 Health Check False Positives

**Symptoms**:
- Unnecessary failovers triggered
- Flapping between regions

**Diagnostics**:

```bash
# Get health check status from all checkers
aws route53 get-health-check-status \
  --health-check-id abc123

# View health check configuration
aws route53 get-health-check \
  --health-check-id abc123
```

**Solutions**:

1. **Adjust failure threshold**
```yaml
HealthCheckConfig:
  Type: HTTPS
  ResourcePath: /health
  FullyQualifiedDomainName: api.example.com
  Port: 443
  RequestInterval: 30  # Check every 30s
  FailureThreshold: 3  # Fail after 3 consecutive failures (90s total)
  # This prevents transient network issues from triggering failover
```

2. **Implement robust health endpoint**
```python
@app.route('/health')
def health_check():
    """
    Comprehensive health check for Route 53.

    Returns 200 only if all critical dependencies are available.
    """
    health = {
        'status': 'healthy',
        'checks': {}
    }

    # Check DynamoDB
    try:
        dynamodb.describe_table(TableName='tickets')
        health['checks']['dynamodb'] = 'ok'
    except:
        health['checks']['dynamodb'] = 'fail'
        health['status'] = 'unhealthy'

    # Check critical Lambda functions
    try:
        lambda_client.get_function(FunctionName='ticket-processor')
        health['checks']['lambda'] = 'ok'
    except:
        health['checks']['lambda'] = 'fail'
        health['status'] = 'unhealthy'

    # Return appropriate status code
    status_code = 200 if health['status'] == 'healthy' else 503

    return jsonify(health), status_code
```

3. **Add CloudWatch alarm for health check**
```yaml
HealthCheckAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: route53-health-check-failing
    MetricName: HealthCheckStatus
    Namespace: AWS/Route53
    Statistic: Minimum
    Period: 60
    EvaluationPeriods: 2  # Alarm after 2 minutes of failures
    Threshold: 1
    ComparisonOperator: LessThanThreshold
    # This gives you visibility before failover triggers
```

## Esempi Reali dal Progetto

### Esempio 1: Ticket Processing Workflow - HA Design

Nel nostro AI Technical Support System, il workflow di processing dei ticket è progettato per alta disponibilità:

```
Client Request
      ↓
API Gateway (Multi-AZ in eu-south-1)
      ↓
Lambda: ticket-ingestion (Multi-AZ)
      ↓
DynamoDB: tickets table (Global Table)
   ├─ eu-south-1 (Primary)
   └─ eu-west-1 (Replica)
      ↓
Step Functions: ticket-processing-workflow
      ↓
  ┌───┴───┬────────┬─────────┐
  │       │        │         │
Lambda: Lambda: Lambda:   Lambda:
Classify Retrieve Generate Validate
(Multi-AZ)(Multi-AZ)(Multi-AZ)(Multi-AZ)
  │       │        │         │
  └───┬───┴────────┴─────────┘
      ↓
DynamoDB: Update ticket status
      ↓
EventBridge: Notify completion
      ↓
SNS: Customer notification
```

**Resilienza Implementata**:

1. **API Gateway**: Regional endpoint con automatic failover
2. **Lambda**: Distribuito automaticamente su 3 AZ
3. **DynamoDB**: Global Tables con replica bi-direzionale
4. **Step Functions**: Retry automatico con exponential backoff
5. **EventBridge**: At-least-once delivery guarantee

**File di Riferimento**: `docs/04-data-flows/ticket-processing.md`

### Esempio 2: Knowledge Base - Cross-Region DR

La knowledge base OpenSearch è critica per il sistema RAG:

```python
# opensearch_dr_manager.py - Dal nostro progetto

class OpenSearchDRManager:
    """Manage OpenSearch disaster recovery."""

    def __init__(self, primary_region='eu-south-1', dr_region='eu-west-1'):
        self.primary_client = OpenSearch(
            hosts=[{'host': f'search-kb-{primary_region}.es.amazonaws.com', 'port': 443}],
            use_ssl=True,
            verify_certs=True,
            connection_class=RequestsHttpConnection,
            region=primary_region
        )

        self.dr_client = OpenSearch(
            hosts=[{'host': f'search-kb-{dr_region}.es.amazonaws.com', 'port': 443}],
            use_ssl=True,
            verify_certs=True,
            connection_class=RequestsHttpConnection,
            region=dr_region
        )

    def create_snapshot(self, repository='s3-snapshot-repo', snapshot_name=None):
        """
        Create snapshot of primary cluster.

        Snapshots are stored in S3 with cross-region replication enabled.
        """
        if not snapshot_name:
            snapshot_name = f"snapshot-{datetime.utcnow().strftime('%Y%m%d-%H%M%S')}"

        response = self.primary_client.snapshot.create(
            repository=repository,
            snapshot=snapshot_name,
            body={
                'indices': 'knowledge-base-*',
                'include_global_state': False,
                'metadata': {
                    'taken_at': datetime.utcnow().isoformat(),
                    'taken_by': 'dr-automation'
                }
            }
        )

        print(f"Snapshot '{snapshot_name}' created: {response}")
        return snapshot_name

    def restore_to_dr(self, snapshot_name, repository='s3-snapshot-repo'):
        """
        Restore snapshot to DR region.

        S3 repository is accessible from DR region via cross-region replication.
        """
        # Register snapshot repository in DR region (if not exists)
        try:
            self.dr_client.snapshot.get_repository(repository=repository)
        except:
            self.dr_client.snapshot.create_repository(
                repository=repository,
                body={
                    'type': 's3',
                    'settings': {
                        'bucket': 'opensearch-snapshots-dr',
                        'region': 'eu-west-1',
                        'role_arn': 'arn:aws:iam::123456789:role/opensearch-snapshot-role'
                    }
                }
            )

        # Restore snapshot
        response = self.dr_client.snapshot.restore(
            repository=repository,
            snapshot=snapshot_name,
            body={
                'indices': 'knowledge-base-*',
                'include_global_state': False,
                'rename_pattern': 'knowledge-base-(.*)',
                'rename_replacement': 'knowledge-base-dr-$1'
            }
        )

        print(f"Restore initiated: {response}")
        return response

    def verify_dr_cluster(self):
        """Verify DR cluster has all necessary data."""
        primary_indices = self.primary_client.cat.indices(format='json')
        dr_indices = self.dr_client.cat.indices(format='json')

        primary_docs = sum(int(idx['docs.count']) for idx in primary_indices if 'knowledge-base' in idx['index'])
        dr_docs = sum(int(idx['docs.count']) for idx in dr_indices if 'knowledge-base' in idx['index'])

        print(f"Primary docs: {primary_docs}")
        print(f"DR docs: {dr_docs}")
        print(f"Difference: {abs(primary_docs - dr_docs)} ({abs(primary_docs - dr_docs) / primary_docs * 100:.2f}%)")

        return {
            'primary_doc_count': primary_docs,
            'dr_doc_count': dr_docs,
            'consistency': (primary_docs - dr_docs) / primary_docs < 0.01  # < 1% difference
        }
```

**Uso nel Progetto**:

```bash
# Automated daily snapshot (cron job)
0 3 * * * /usr/local/bin/python /opt/scripts/opensearch_snapshot.py

# Manual DR test (quarterly)
python opensearch_dr_manager.py --action test-failover
```

**File di Riferimento**: `docs/03-aws-services/opensearch.md`

### Esempio 3: SageMaker Endpoint - Multi-AZ Deployment

Il nostro classificatore di ticket usa SageMaker:

```python
# sagemaker_ha_deployment.py

import boto3
import sagemaker
from sagemaker.model import Model

class HAEndpointDeployer:
    """Deploy SageMaker endpoint with HA configuration."""

    def __init__(self, model_name='ticket-classifier', region='eu-south-1'):
        self.sagemaker_client = boto3.client('sagemaker', region_name=region)
        self.model_name = model_name
        self.region = region

    def create_endpoint_config(self, instance_type='ml.m5.xlarge', instance_count=2):
        """
        Create endpoint configuration with multi-AZ deployment.

        Args:
            instance_type: EC2 instance type for hosting
            instance_count: Minimum 2 for HA across AZs
        """
        config_name = f"{self.model_name}-config-{int(time.time())}"

        self.sagemaker_client.create_endpoint_config(
            EndpointConfigName=config_name,
            ProductionVariants=[{
                'VariantName': 'AllTraffic',
                'ModelName': self.model_name,
                'InstanceType': instance_type,
                'InitialInstanceCount': instance_count,  # Distributed across AZs
                'InitialVariantWeight': 1.0,

                # Auto-scaling configuration
                'AutoScalingConfiguration': {
                    'MinCapacity': instance_count,
                    'MaxCapacity': instance_count * 3,
                    'TargetTrackingScalingPolicyConfiguration': {
                        'TargetValue': 70.0,  # 70% invocations per instance
                        'PredefinedMetricSpecification': {
                            'PredefinedMetricType': 'SageMakerVariantInvocationsPerInstance'
                        },
                        'ScaleInCooldown': 600,  # 10 min
                        'ScaleOutCooldown': 300  # 5 min
                    }
                }
            }],

            # Data capture for model monitoring
            DataCaptureConfig={
                'EnableCapture': True,
                'InitialSamplingPercentage': 100,
                'DestinationS3Uri': f's3://sagemaker-data-capture/{self.model_name}/',
                'CaptureOptions': [
                    {'CaptureMode': 'Input'},
                    {'CaptureMode': 'Output'}
                ]
            },

            Tags=[
                {'Key': 'Environment', 'Value': 'production'},
                {'Key': 'HA', 'Value': 'true'}
            ]
        )

        print(f"Created endpoint config: {config_name}")
        return config_name

    def deploy_with_blue_green(self, new_config_name):
        """
        Blue/green deployment for zero-downtime updates.

        Gradually shifts traffic from old to new configuration.
        """
        endpoint_name = f"{self.model_name}-endpoint"

        # Update endpoint to use new configuration
        self.sagemaker_client.update_endpoint(
            EndpointName=endpoint_name,
            EndpointConfigName=new_config_name,
            DeploymentConfig={
                'BlueGreenUpdatePolicy': {
                    'TrafficRoutingConfiguration': {
                        'Type': 'CANARY',
                        'CanarySize': {
                            'Type': 'CAPACITY_PERCENT',
                            'Value': 10  # Start with 10% traffic
                        },
                        'WaitIntervalInSeconds': 300  # Wait 5 minutes
                    },
                    'TerminationWaitInSeconds': 600,  # Keep old for 10 min
                    'MaximumExecutionTimeoutInSeconds': 3600
                },
                'AutoRollbackConfiguration': {
                    'Alarms': [{
                        'AlarmName': f'{self.model_name}-model-error-rate'
                    }]
                }
            },
            RetainAllVariantProperties=False
        )

        print(f"Blue/green deployment initiated for {endpoint_name}")
```

**Metrics e Alarms**:

```python
def setup_endpoint_alarms(endpoint_name):
    """Setup CloudWatch alarms for endpoint health."""
    cloudwatch = boto3.client('cloudwatch')

    # High error rate alarm
    cloudwatch.put_metric_alarm(
        AlarmName=f'{endpoint_name}-high-error-rate',
        MetricName='ModelInvocation5XXErrors',
        Namespace='AWS/SageMaker',
        Statistic='Sum',
        Period=300,
        EvaluationPeriods=2,
        Threshold=10,
        ComparisonOperator='GreaterThanThreshold',
        Dimensions=[
            {'Name': 'EndpointName', 'Value': endpoint_name},
            {'Name': 'VariantName', 'Value': 'AllTraffic'}
        ],
        TreatMissingData='notBreaching'
    )

    # High latency alarm
    cloudwatch.put_metric_alarm(
        AlarmName=f'{endpoint_name}-high-latency',
        MetricName='ModelLatency',
        Namespace='AWS/SageMaker',
        Statistic='Average',
        Period=300,
        EvaluationPeriods=2,
        Threshold=5000,  # 5 seconds
        ComparisonOperator='GreaterThanThreshold',
        Dimensions=[
            {'Name': 'EndpointName', 'Value': endpoint_name},
            {'Name': 'VariantName', 'Value': 'AllTraffic'}
        ]
    )

    print(f"Alarms configured for {endpoint_name}")
```

**File di Riferimento**: `docs/03-aws-services/sagemaker.md`

## Riferimenti

### Documentazione Interna del Progetto

- [Architecture Overview](../02-architecture/overview.md) - Panoramica architettura sistema
- [Deployment Topology](../02-architecture/deployment.md) - Dettagli deployment multi-AZ
- [Security Architecture](../02-architecture/security.md) - Best practices sicurezza
- [AWS Services](../03-aws-services/README.md) - Configurazione servizi AWS
- [Data Models](../06-data-models.md) - Schema DynamoDB e OpenSearch
- [Ticket Processing Flow](../04-data-flows/ticket-processing.md) - Workflow completo
- [Cost Estimation](../11-cost-estimation.md) - Analisi costi DR/HA

### AWS Documentation

#### High Availability
- [AWS Well-Architected Framework - Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [Building Multi-AZ Applications](https://aws.amazon.com/architecture/reference-architecture-diagrams/?solutions-all.sort-by=item.additionalFields.sortDate&solutions-all.sort-order=desc&whitepapers-main.sort-by=item.additionalFields.sortDate&whitepapers-main.sort-order=desc&awsf.whitepapers-tech-category=tech-category%23databases)
- [Amazon RDS Multi-AZ Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [DynamoDB Global Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)
- [Lambda Function Resilience](https://docs.aws.amazon.com/lambda/latest/dg/lambda-resilience.html)

#### Disaster Recovery
- [AWS Disaster Recovery Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
- [S3 Cross-Region Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [Route 53 Health Checks and Failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html)

#### Service-Specific
- [OpenSearch Snapshots](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-snapshots.html)
- [SageMaker Multi-Model Endpoints](https://docs.aws.amazon.com/sagemaker/latest/dg/multi-model-endpoints.html)
- [Step Functions Error Handling](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)

### Blog Posts e Articoli

- [AWS Architecture Blog - Disaster Recovery](https://aws.amazon.com/blogs/architecture/tag/disaster-recovery/)
- [Chaos Engineering on AWS](https://aws.amazon.com/blogs/architecture/verify-the-resilience-of-your-workloads-using-chaos-engineering/)
- [Multi-Region Serverless Patterns](https://aws.amazon.com/blogs/compute/building-a-multi-region-serverless-application-with-amazon-api-gateway-and-aws-lambda/)

### Tools e Utilities

- [AWS Fault Injection Simulator](https://aws.amazon.com/fis/) - Chaos engineering testing
- [AWS Resilience Hub](https://aws.amazon.com/resilience-hub/) - Application resilience assessment
- [CloudFormation StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html) - Multi-region deployment
- [AWS Systems Manager - Incident Manager](https://aws.amazon.com/systems-manager/features/#Incident_Manager) - Incident response automation

### Community Resources

- [AWS re:Post - Disaster Recovery](https://repost.aws/tags/TA4IHBVReRRl-dds-1OdF8oQ/disaster-recovery)
- [AWS Samples - Disaster Recovery](https://github.com/aws-samples?q=disaster-recovery)
- [AWS Solutions Library](https://aws.amazon.com/solutions/)

---

**Maintainer**: Platform Engineering Team
**Last Updated**: 2025-11-18
**Next Review**: 2026-02-18
**Version**: 1.0

