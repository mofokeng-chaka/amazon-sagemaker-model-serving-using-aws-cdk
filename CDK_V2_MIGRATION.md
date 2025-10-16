# CDK v1 to v2 Migration Guide

This document outlines the steps taken to migrate this SageMaker model serving project from AWS CDK v1 to v2.

## Migration Steps

### 1. Update package.json

Replace CDK v1 dependencies with v2 equivalents:

```json
{
  "devDependencies": {
    "aws-cdk": "2.x"
  },
  "dependencies": {
    "aws-cdk-lib": "2.x",
    "constructs": "^10.0.0"
  }
}
```

Remove all individual `@aws-cdk/*` packages - they're now bundled in `aws-cdk-lib`.

### 2. Update Import Statements

**Before (CDK v1):**
```typescript
import * as cdk from '@aws-cdk/core';
import * as lambda from '@aws-cdk/aws-lambda';
import * as s3 from '@aws-cdk/aws-s3';
```

**After (CDK v2):**
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as s3 from 'aws-cdk-lib/aws-s3';
```

### 3. Update Construct References

**Before:**
```typescript
export class MyStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
```

**After:**
```typescript
export class MyStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
```

### 4. Fix Managed Policy References

**Before:**
```typescript
managedPolicies: [
  { managedPolicyArn: 'arn:aws:iam::aws:policy/AmazonSageMakerFullAccess' }
]

role.addManagedPolicy({ managedPolicyArn: 'arn:aws:iam::aws:policy/AmazonS3FullAccess' });
```

**After:**
```typescript
managedPolicies: [
  iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonSageMakerFullAccess')
]

role.addManagedPolicy(iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonS3FullAccess'));
```

### 5. Fix Lambda Alias Usage

**Before (deprecated):**
```typescript
const lambdaInferAlias = lambdaFunction.currentVersion.addAlias(stageName);
```

**After (recommended):**
```typescript
const lambdaInferAlias = lambdaFunction.addAlias(stageName);
```

### 6. Update Lambda Runtime

**Before (deprecated):**
```typescript
runtime: lambda.Runtime.PYTHON_3_7,
```

**After (supported):**
```typescript
runtime: lambda.Runtime.PYTHON_3_11,
```

### 7. Fix S3 Bucket Naming

**Before (too long):**
```typescript
const suffix: string = `${this.commonProps.env?.region}-${this.commonProps.env?.account?.substr(0, 5)}-${Date.now()}`
```

**After (shorter):**
```typescript
const randomSuffix = Math.random().toString(36).substring(2, 8);
const suffix: string = `${this.commonProps.env?.region}-${randomSuffix}`
```

### 8. Clean Up cdk.json

Remove deprecated CDK v1 feature flags:

**Before:**
```json
{
  "context": {
    "@aws-cdk/core:enableStackNameDuplicates": "true",
    "@aws-cdk/core:stackRelativeExports": "true",
    "@aws-cdk/aws-ecr-assets:dockerIgnoreSupport": true
  }
}
```

**After:**
```json
{
  "context": {
    "aws-cdk:enableDiffNoFail": "true"
  }
}
```

## Automated Migration Commands

Run these commands to perform the migration:

```bash
# Update core imports
find bin lib -name "*.ts" -exec sed -i '' 's/@aws-cdk\/core/aws-cdk-lib/g' {} \;

# Update service imports
find bin lib -name "*.ts" -exec sed -i '' 's/@aws-cdk\/aws-/aws-cdk-lib\/aws-/g' {} \;

# Add Construct import and fix references
find bin lib -name "*.ts" -exec sed -i '' 's/import \* as cdk from '\''aws-cdk-lib'\'';/import * as cdk from '\''aws-cdk-lib'\'';\nimport { Construct } from '\''constructs'\'';/g' {} \;
find bin lib -name "*.ts" -exec sed -i '' 's/cdk\.Construct/Construct/g' {} \;

# Fix managed policies
find bin lib -name "*.ts" -exec sed -i '' 's/{ managedPolicyArn: '\''arn:aws:iam::aws:policy\/\([^'\'']*\)'\'' }/iam.ManagedPolicy.fromAwsManagedPolicyName('\''\1'\'')/g' {} \;
find bin lib -name "*.ts" -exec sed -i '' 's/addManagedPolicy({ managedPolicyArn: '\''arn:aws:iam::aws:policy\/\([^'\'']*\)'\'' })/addManagedPolicy(iam.ManagedPolicy.fromAwsManagedPolicyName('\''\1'\''))/g' {} \;

# Fix Lambda alias usage
find bin lib -name "*.ts" -exec sed -i '' 's/\.currentVersion\.addAlias(/\.addAlias(/g' {} \;

# Update Lambda runtime
find bin lib -name "*.ts" -exec sed -i '' 's/lambda\.Runtime\.PYTHON_3_7/lambda.Runtime.PYTHON_3_11/g' {} \;
```

## Verification

After migration:

1. **Install dependencies:**
   ```bash
   npm install
   ```

2. **Build the project:**
   ```bash
   npm run build
   ```

3. **Bootstrap CDK (with valid AWS credentials):**
   ```bash
   npx cdk bootstrap
   ```

4. **Deploy stacks in dependency order:**
   ```bash
   npx cdk deploy TextClassificationDemo-ModelArchivingStack
   npx cdk deploy TextClassificationDemo-ModelServingStack
   npx cdk deploy TextClassificationDemo-APIHostingStack
   npx cdk deploy TextClassificationDemo-MonitorDashboardStack
   ```

   **Note:** Deploy stacks sequentially as they have dependencies on SSM parameters from previous stacks.

## Key Differences in CDK v2

- **Single package:** All AWS services are in `aws-cdk-lib`
- **Construct import:** `Construct` comes from `constructs` package
- **Managed policies:** Use `ManagedPolicy.fromAwsManagedPolicyName()` instead of ARN objects
- **Lambda aliases:** Use `function.addAlias()` instead of `version.addAlias()`
- **Lambda runtime:** Updated from deprecated Python 3.7 to Python 3.11
- **S3 bucket naming:** Fixed length issues with shorter random suffixes
- **Feature flags:** Many v1 flags are deprecated and removed
- **Breaking changes:** Some APIs have changed - check CDK v2 migration guide for specifics

## Troubleshooting

**Authentication Error:**
- Ensure AWS credentials are configured and not expired
- Run `aws configure list` to verify credentials
- Use `aws sso login` if using AWS SSO

**Build Errors:**
- Check for remaining `@aws-cdk/` imports
- Verify all `cdk.Construct` references are changed to `Construct`
- Ensure `constructs` package is installed

**Lambda Runtime Errors:**
- Update deprecated Python 3.7 to Python 3.11 or newer
- Check AWS Lambda supported runtimes documentation

**S3 Bucket Name Errors:**
- Ensure bucket names are under 63 characters
- Use shorter suffixes or random strings for uniqueness

**Deprecated Warnings:**
- Review CDK v2 documentation for updated APIs
- Replace deprecated methods with recommended alternatives
