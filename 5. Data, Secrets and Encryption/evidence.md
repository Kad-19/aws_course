


# Lab Evidence: Data, Secrets, and Encryption

## 1. Architecture Flow
* **Network Route:** Lambda connects to the RDS Proxy endpoint on port 5432 entirely within the private, isolated subnets.
* **Secret Retrieval:** RDS Proxy leverages a VPC Interface Endpoint to securely connect to AWS Secrets Manager without using the public internet. 
* **Decryption:** Secrets Manager utilizes AWS KMS to decrypt the stored database password.
* **Database Connection:** RDS Proxy uses the decrypted physical password to authenticate and establish a connection pool with the Aurora PostgreSQL database.

## 2. Required Permissions
* **IAM Permissions:** 
    * Lambda requires `rds-db:connect` to generate the authentication token.
    * RDS Proxy requires `secretsmanager:GetSecretValue` and `kms:Decrypt` to fetch the physical password.
* **Network Permissions:** 
    * Lambda Security Group requires outbound access to RDS Proxy on port 5432.
    * RDS Proxy Security Group requires inbound access from Lambda on port 5432, and outbound access to Aurora on port 5432.
    * VPC Interface Endpoints require inbound access from the RDS Proxy Security Group on port 443 (HTTPS) to retrieve the secret.
* **Database Permissions:** 
    * Aurora requires a database user (`labadmin`) that matches the credentials stored in Secrets Manager.

## 3. CDK Diff Output
*Note: The following `DataSecretsLabStack` stack output is from a new lab created for practice.*

### Stack `DataSecretsLabStack`

#### IAM Statement Changes

| +/- | Resource | Effect | Action | Principal | Condition |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **+** | `${DbInsertFunction/ServiceRole.Arn}` | Allow | `sts:AssumeRole` | `Service:lambda.amazonaws.com` | |
| **+** | `${DbKey.Arn}` | Allow | `kms:*` | `AWS:arn:${AWS::Partition}:iam::${AWS::AccountId}:root` | |
| **+** | `${DbKey.Arn}` | Allow | `kms:Decrypt`<br>`kms:Encrypt`<br>`kms:GenerateDataKey*`<br>`kms:ReEncrypt*` | `AWS:arn:${AWS::Partition}:iam::${AWS::AccountId}:root` | `"StringEquals": { "kms:ViaService": "secretsmanager.${AWS::Region}.amazonaws.com" }` |
| **+** | `${DbKey.Arn}` | Allow | `kms:CreateGrant`<br>`kms:DescribeKey` | `AWS:arn:${AWS::Partition}:iam::${AWS::AccountId}:root` | `"StringEquals": { "kms:ViaService": "secretsmanager.${AWS::Region}.amazonaws.com" }` |
| **+** | `${DbKey.Arn}` | Allow | `kms:Decrypt` | `AWS:${LabProxy/IAMRole.Arn}` | `"StringEquals": { "kms:ViaService": "secretsmanager.${AWS::Region}.amazonaws.com" }` |
| **+** | `${DbKey.Arn}` | Allow | `kms:Decrypt` | `AWS:${LabProxy/IAMRole}` | |
| **+** | `${DbSecret}` | Allow | `secretsmanager:DescribeSecret`<br>`secretsmanager:GetSecretValue` | `AWS:${LabProxy/IAMRole}` | |
| **+** | `${DbTestFunction/ServiceRole.Arn}` | Allow | `sts:AssumeRole` | `Service:lambda.amazonaws.com` | |
| **+** | `${LabProxy/IAMRole.Arn}` | Allow | `sts:AssumeRole` | `Service:rds.amazonaws.com` | |
| **+** | `arn:${AWS::Partition}:rds-db:${AWS::Region}:${AWS::AccountId}:dbuser:{"Fn::Select":[6,"{\"Fn::Split\":[\":\",\"${LabProxy7456B4C3.DBProxyArn}\"]}"]}/labadmin` | Allow | `rds-db:connect` | `AWS:${DbTestFunction/ServiceRole}` | |
| **+** | `arn:${AWS::Partition}:rds-db:${AWS::Region}:${AWS::AccountId}:dbuser:{"Fn::Select":[6,"{\"Fn::Split\":[\":\",\"${LabProxy7456B4C3.DBProxyArn}\"]}"]}/labadmin` | Allow | `rds-db:connect` | `AWS:${DbInsertFunction/ServiceRole}` | |

#### IAM Policy Changes

| +/- | Resource | Managed Policy ARN |
| :--- | :--- | :--- |
| **+** | `${DbInsertFunction/ServiceRole}` | `arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole` |
| **+** | `${DbInsertFunction/ServiceRole}` | `arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole` |
| **+** | `${DbTestFunction/ServiceRole}` | `arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole` |
| **+** | `${DbTestFunction/ServiceRole}` | `arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole` |

#### Security Group Changes

| +/- | Group | Dir | Protocol | Peer |
| :--- | :--- | :--- | :--- | :--- |
| **+** | `${DbInsertFunction/SecurityGroup.GroupId}` | Out | Everything | Everyone (IPv4) |
| **+** | `${DbTestFunction/SecurityGroup.GroupId}` | Out | Everything | Everyone (IPv4) |
| **+** | `${LabCluster/SecurityGroup.GroupId}` | In | TCP `${LabCluster.Endpoint.Port}` | `${LabProxy/ProxySecurityGroup.GroupId}` |
| **+** | `${LabCluster/SecurityGroup.GroupId}` | Out | Everything | Everyone (IPv4) |
| **+** | `${LabProxy/ProxySecurityGroup.GroupId}` | In | TCP 5432 | `${DbTestFunction/SecurityGroup.GroupId}` |
| **+** | `${LabProxy/ProxySecurityGroup.GroupId}` | In | TCP 5432 | `${DbInsertFunction/SecurityGroup.GroupId}` |
| **+** | `${LabProxy/ProxySecurityGroup.GroupId}` | Out | Everything | Everyone (IPv4) |

*(NOTE: There may be security-related changes not in this list. See https://github.com/aws/aws-cdk/issues/1299)*

---

#### Parameters
* **[+]** `Parameter BootstrapVersion BootstrapVersion:` 
  ```json
  {"Type":"AWS::SSM::Parameter::Value<String>","Default":"/cdk-bootstrap/hnb659fds/version","Description":"Version of the CDK Bootstrap resources in this environment, automatically retrieved from SSM Parameter Store. [cdk:skip]"}
#### Conditions
* **[+]** `Condition CDKMetadata/Condition CDKMetadataAvailable:` `{"Fn::Or":[{"Fn::Or":[{"Fn::Equals":[{"Ref":"AWS::Region"},"af-south-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-east-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-northeast-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-northeast-2"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-northeast-3"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-south-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-south-2"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-southeast-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-southeast-2"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-southeast-3"]}]},{"Fn::Or":[{"Fn::Equals":[{"Ref":"AWS::Region"},"ap-southeast-4"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ca-central-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"ca-west-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"cn-north-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"cn-northwest-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"eu-central-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"eu-central-2"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"eu-north-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"eu-south-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"eu-south-2"]}]},{"Fn::Or":[{"Fn::Equals":[{"Ref":"AWS::Region"},"eu-west-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"eu-west-2"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"eu-west-3"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"il-central-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"me-central-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"me-south-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"sa-east-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"us-east-1"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"us-east-2"]},{"Fn::Equals":[{"Ref":"AWS::Region"},"us-west-1"]}]},{"Fn::Equals":[{"Ref":"AWS::Region"},"us-west-2"]}]}`

#### Resources
* **[+]** `AWS::EC2::VPC` LabVpc LabVpc17F821B7
* **[+]** `AWS::EC2::Subnet` LabVpc/PublicSubnet1/Subnet LabVpcPublicSubnet1SubnetBE5AC483
* **[+]** `AWS::EC2::RouteTable` LabVpc/PublicSubnet1/RouteTable LabVpcPublicSubnet1RouteTable5D022469
* **[+]** `AWS::EC2::SubnetRouteTableAssociation` LabVpc/PublicSubnet1/RouteTableAssociation LabVpcPublicSubnet1RouteTableAssociationFE601F86
* **[+]** `AWS::EC2::Route` LabVpc/PublicSubnet1/DefaultRoute LabVpcPublicSubnet1DefaultRouteD07C003C
* **[+]** `AWS::EC2::Subnet` LabVpc/PublicSubnet2/Subnet LabVpcPublicSubnet2Subnet17E729C0
* **[+]** `AWS::EC2::RouteTable` LabVpc/PublicSubnet2/RouteTable LabVpcPublicSubnet2RouteTableA796C712
* **[+]** `AWS::EC2::SubnetRouteTableAssociation` LabVpc/PublicSubnet2/RouteTableAssociation LabVpcPublicSubnet2RouteTableAssociationD6683741
* **[+]** `AWS::EC2::Route` LabVpc/PublicSubnet2/DefaultRoute LabVpcPublicSubnet2DefaultRouteC6E99F9A
* **[+]** `AWS::EC2::Subnet` LabVpc/IsolatedSubnet1/Subnet LabVpcIsolatedSubnet1Subnet3215AE8D
* **[+]** `AWS::EC2::RouteTable` LabVpc/IsolatedSubnet1/RouteTable LabVpcIsolatedSubnet1RouteTableDB994D83
* **[+]** `AWS::EC2::SubnetRouteTableAssociation` LabVpc/IsolatedSubnet1/RouteTableAssociation LabVpcIsolatedSubnet1RouteTableAssociationD6F59C2B
* **[+]** `AWS::EC2::Subnet` LabVpc/IsolatedSubnet2/Subnet LabVpcIsolatedSubnet2SubnetF05A8759
* **[+]** `AWS::EC2::RouteTable` LabVpc/IsolatedSubnet2/RouteTable LabVpcIsolatedSubnet2RouteTable43352418
* **[+]** `AWS::EC2::SubnetRouteTableAssociation` LabVpc/IsolatedSubnet2/RouteTableAssociation LabVpcIsolatedSubnet2RouteTableAssociation83FF6F31
* **[+]** `AWS::EC2::InternetGateway` LabVpc/IGW LabVpcIGW82336A21
* **[+]** `AWS::EC2::VPCGatewayAttachment` LabVpc/VPCGW LabVpcVPCGW932EA6D0
* **[+]** `AWS::KMS::Key` DbKey DbKeyD09445D3
* **[+]** `AWS::SecretsManager::Secret` DbSecret DbSecret685A0FA5
* **[+]** `AWS::SecretsManager::SecretTargetAttachment` DbSecret/Attachment DbSecretAttachment0609CE05
* **[+]** `AWS::RDS::DBSubnetGroup` LabCluster/Subnets LabClusterSubnets45F93620
* **[+]** `AWS::EC2::SecurityGroup` LabCluster/SecurityGroup LabClusterSecurityGroup54110535
* **[+]** `AWS::EC2::SecurityGroupIngress` LabCluster/SecurityGroup/from DataSecretsLabStackLabProxyProxySecurityGroupA67202B6:{IndirectPort} LabClusterSecurityGroupfromDataSecretsLabStackLabProxyProxySecurityGroupA67202B6IndirectPort4F101333
* **[+]** `AWS::RDS::DBCluster` LabCluster LabClusterA4B25836
* **[+]** `AWS::RDS::DBInstance` LabCluster/writer LabClusterwriterD49A3AAD
* **[+]** `AWS::IAM::Role` LabProxy/IAMRole LabProxyIAMRoleC363F25D
* **[+]** `AWS::IAM::Policy` LabProxy/IAMRole/DefaultPolicy LabProxyIAMRoleDefaultPolicy86CBF0D8
* **[+]** `AWS::EC2::SecurityGroup` LabProxy/ProxySecurityGroup LabProxyProxySecurityGroupC0AC4333
* **[+]** `AWS::EC2::SecurityGroupIngress` LabProxy/ProxySecurityGroup/from DataSecretsLabStackDbTestFunctionSecurityGroup161A2A1C:5432 LabProxyProxySecurityGroupfromDataSecretsLabStackDbTestFunctionSecurityGroup161A2A1C5432059A6656
* **[+]** `AWS::EC2::SecurityGroupIngress` LabProxy/ProxySecurityGroup/from DataSecretsLabStackDbInsertFunctionSecurityGroupB37B6C34:5432 LabProxyProxySecurityGroupfromDataSecretsLabStackDbInsertFunctionSecurityGroupB37B6C345432C9D1E0DB
* **[+]** `AWS::RDS::DBProxy` LabProxy LabProxy7456B4C3
* **[+]** `AWS::RDS::DBProxyTargetGroup` LabProxy/ProxyTargetGroup LabProxyProxyTargetGroup460393E9
* **[+]** `AWS::IAM::Role` DbTestFunction/ServiceRole DbTestFunctionServiceRoleC698218C
* **[+]** `AWS::IAM::Policy` DbTestFunction/ServiceRole/DefaultPolicy DbTestFunctionServiceRoleDefaultPolicy7E621BD1
* **[+]** `AWS::EC2::SecurityGroup` DbTestFunction/SecurityGroup DbTestFunctionSecurityGroup665D3DB0
* **[+]** `AWS::Lambda::Function` DbTestFunction DbTestFunction8C941921
* **[+]** `AWS::IAM::Role` DbInsertFunction/ServiceRole DbInsertFunctionServiceRole60512DDF
* **[+]** `AWS::IAM::Policy` DbInsertFunction/ServiceRole/DefaultPolicy DbInsertFunctionServiceRoleDefaultPolicy73E5EA0F
* **[+]** `AWS::EC2::SecurityGroup` DbInsertFunction/SecurityGroup DbInsertFunctionSecurityGroup3AB1A66F
* **[+]** `AWS::Lambda::Function` DbInsertFunction DbInsertFunction52A32E09

#### Outputs
* **[+]** `Output InsertLambdaFunctionName InsertLambdaFunctionName:` `{"Value":{"Ref":"DbInsertFunction52A32E09"}}`
* **[+]** `Output ProxyEndpoint ProxyEndpoint:` `{"Value":{"Fn::GetAtt":["LabProxy7456B4C3","Endpoint"]}}`
* **[+]** `Output LambdaFunctionName LambdaFunctionName:` `{"Value":{"Ref":"DbTestFunction8C941921"}}`

## 4. Successful Test Query Outputs
**Test Query Output:**
Lab was CDK-diff-only. Physical deployment and Lambda invocations were skipped due to AWS Free Tier constraints on the Amazon RDS Proxy service.

## 5. Cleanup Confirmation
Lab was CDK-diff-only. No physical AWS resources were deployed, so no teardown or cleanup was required.