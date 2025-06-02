---
published: false
title: Deploy a Global Serverless API with the CDK
cover_image: TODO
description: TODO
tags: Typescript, Serverless, AWS, DynamoDB, API Gateway, Lambda, CDK
---

In today’s digital world, users expect fast and reliable application performance no matter where they are. To meet these expectations, deploying global infrastructure is often necessary.

This article walks you through building and deploying a **globally available serverless API** with **Amazon API Gateway**, **AWS Lambda**, and **DynamoDB Global Tables** using the **AWS CDK**.

## 🧱 Architecture Overview

Here's the architecture we’ll deploy:

- **Amazon API Gateway**: Public HTTP API for handling client requests.
- **AWS Lambda**: Stateless backend function to process requests.
- **Amazon DynamoDB Global Tables**: Fully replicated database across multiple AWS regions for low-latency reads and writes.
- **AWS CDK**: Infrastructure as code to manage and deploy everything.

Schema here

We'll assume you have a top domain.

## 📦 Step 1: Set Up Your CDK App

First, set up a new CDK app:

```bash
# Create the folder
mkdir global-api-cdk && cd global-api-cdk
# Init the CDK App
npx cdk init app --language typescript
# Add some dependencies
npm add @types/aws-lambda @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

## 🌍 Step 2: Choose Your Regions

Pick:

- One AWS region to host your global resources like DynamoDB table (e.g. `us-east-1`)
- One or more AWS regions to replicate your API, Lambda functions and data (e.g. `eu-west-1`, `sa-east-1`...)

```ts
// lib/regions.ts
export const globalResourcesRegion = 'us-east-1';
export const replicatedRegions = ['eu-west-1', 'sa-east-1'];

export const allRegions = [globalResourcesRegion, ...replicatedRegions];
```

Each replicated region will host a replica of the API and DynamoDB table. The Global Table ensures data sync across all regions.

## 🏗️ Step 3: Define a DynamoDB Global Table

Create a `GlobalResourcesStack` containing a `TableV2` with `replicationRegions` specified:

```ts
// lib/global.ts
import { Construct } from 'constructs';
import { Stack, StackProps } from 'aws-cdk-lib';
import { AttributeType, TableV2 } from 'aws-cdk-lib/aws-dynamodb';

import { replicatedRegions } from './regions';

export class GlobalResourcesStack extends Stack {
  table: TableV2;

  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    this.table = new TableV2(this, 'Table', {
      tableName: 'global-table',
      partitionKey: { name: 'id', type: AttributeType.STRING },
      replicas: replicatedRegions.map(region => ({ region })),
    });
  }
}
```

> Note: The table must be created in a single region first and then replicated.

```ts
// lib/local.ts
import { Construct } from 'constructs';
import { Stack, StackProps } from 'aws-cdk-lib';
import { ITableV2, TableV2 } from 'aws-cdk-lib/aws-dynamodb';

import { GlobalResourcesStack } from './global';

export class LocalResourcesStack extends Stack {
  // 👇 Use ITableV2 type as the table is in the global stack
  table: ITableV2;

  constructor(
    scope: Construct,
    id: string,
    {
      globalResources,
      ...props
    }: { globalResources: GlobalResourcesStack } & StackProps,
  ) {
    super(scope, id, props);

    // 👇 Use `fromTableName` to replace the stack region
    this.table = TableV2.fromTableName(
      this,
      'Table',
      globalResources.table.tableName,
    );
  }
}
```

Finaly we can define our CDK App like this:

```ts
// bin/global-api-cdk.ts
import { App } from 'aws-cdk-lib';

import { allRegions, globalResourcesRegion } from '../lib/regions';
import { GlobalResourcesStack } from '../lib/global';
import { LocalResourcesStack } from '../lib/local';

const app = new App();

const globalResources = new GlobalResourcesStack(app, 'GlobalResources', {
  env: { region: globalResourcesRegion },
  // 👇 Important to pass References from a stack to another
  crossRegionReferences: true,
});

for (const region of allRegions) {
  const localResources = new LocalResourcesStack(
    app,
    // Stack names need to be unique
    `LocalResources${region}`,
    {
      env: { region },
      globalResources,
      crossRegionReferences: region !== globalResourcesRegion,
    },
  );

  localResources.addDependency(globalResources);
}
```

## 🧠 Step 4: Create API Gateways

This App is still incomplete: Let's keep it rolling by adding an API.

We'll begin by importing the Hosted Zone (i.e. `example.com`) within our Global Stack, and defining a subdomain hosted zone for our api (`api.example.com`).

Hosted zones are global, so let's add it to the Global Resources Stack:

```ts
import {
  IHostedZone,
  HostedZone,
  RecordSet,
  RecordTarget,
  RecordType,
} from 'aws-cdk-lib/aws-route53';

class GlobalResourcesStack extends Stack {
  // ...
  topDomainHostedZone: IHostedZone;
  apiSubDomainHostedZone: HostedZone;
  apiSubDomainDelegationRecords: RecordSet;

  constructor(...a) {
    // ...

    this.topDomainHostedZone = HostedZone.fromHostedZoneAttributes(
      this,
      'TopDomainHostedZone',
      // 👇 Imported from your Top Domain configuration
      { zoneName: 'example.com', hostedZoneId: 'Z31867814UKQT2MAKXUTL' },
    );

    this.apiSubDomainHostedZone = new HostedZone(
      this,
      'ApiSubDomainHostedZone',
      { zoneName: 'api.example.com' },
    );

    // 👇 (optional) To automatically append the delegation records
    this.apiSubDomainDelegationRecords = new RecordSet(
      this,
      'ApiSubDomainDelegationRecords',
      {
        zone: this.topDomainHostedZone,
        recordType: RecordType.NS,
        recordName: this.apiSubDomainHostedZone.zoneName,
        target: RecordTarget.fromValues(
          ...this.apiSubDomainHostedZone.hostedZoneNameServers!,
        ),
      },
    );
  }
}
```

Let's import and use it as a custom domain for an API with a certificate:

```ts
import { IHostedZone } from 'aws-cdk-lib/aws-route53';
import {
  Certificate,
  CertificateValidation,
} from 'aws-cdk-lib/aws-certificatemanager';
import { RestApi } from 'aws-cdk-lib/aws-apigateway';

class LocalResourcesStack extends Stack {
  // ...
  // 👇 Use IHostedZone type as the HostedZone is in the global stack
  apiSubDomainHostedZone: IHostedZone;
  apiCertificate: Certificate;
  api: RestApi;

  constructor(...a) {
    // ...
    this.apiSubDomainHostedZone = globalResources.apiSubDomainHostedZone;

    this.apiCertificate = new Certificate(this, 'ApiCertificate', {
      domainName: this.apiSubDomainHostedZone.zoneName,
      validation: CertificateValidation.fromDns(this.apiSubDomainHostedZone),
    });

    this.api = new RestApi(this, 'Api', {
      domainName: {
        domainName: this.apiSubDomainHostedZone.zoneName,
        certificate: this.apiCertificate,
      },
    });
  }
}
```

## 🧠 Step 5: Create Latency Records

First we need to create a [Latency Record](#todo). This small hack is the only way to set Routing Policy to Latency with the CDK for now (See https://github.com/aws/aws-cdk/issues/4391):

```ts
// lib/latencyRecord.ts
import { ARecordProps, CfnRecordSet, ARecord } from 'aws-cdk-lib/aws-route53';
import { Construct } from 'constructs';

export class LatencyRecord extends ARecord {
  constructor(
    scope: Construct,
    id: string,
    { region, ...props }: { region: string } & ARecordProps,
  ) {
    super(scope, id, props);

    const recordSet = this.node.defaultChild as CfnRecordSet;
    recordSet.region = region;
    recordSet.setIdentifier = region;
  }
}
```

Now let's use it:

```ts
import { ApiGateway } from 'aws-cdk-lib/aws-route53-targets';
import { RecordTarget } from 'aws-cdk-lib/aws-route53';

import { LatencyRecord } from './latencyRecord';

class LocalResourcesStack extends Stack {
  // ...

  constructor(...a) {
    // ...

    new LatencyRecord(this, 'ApiSubDomainLatencyRecord', {
      zone: this.apiSubDomainHostedZone,
      target: RecordTarget.fromAlias(new ApiGateway(this.api)),
      region: this.region,
    });
  }
}
```

## 🧠 Step 6: Create a Lambda Function to Access the Table

Finally, let's add a Lambda Function in the Local Stack that accesses our Global Table:

```ts
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { Runtime, Architecture } from 'aws-cdk-lib/aws-lambda';
import { LogGroup, RetentionDays } from 'aws-cdk-lib/aws-logs';
import { LambdaIntegration } from 'aws-cdk-lib/aws-apigateway';

class LocalResourcesStack extends Stack {
  // ...
  lambda: NodejsFunction;

  constructor(...a) {
    // ...

    this.lambda = new NodejsFunction(this, 'Function', {
      entry: 'lib/handler.ts',
      handler: 'main',
      environment: {
        TABLE_NAME: this.table.tableName,
      },
      runtime: Runtime.NODEJS_20_X,
      architecture: Architecture.ARM_64,
      awsSdkConnectionReuse: true,
      logGroup: new LogGroup(this, 'FunctionLogGroup', {
        logGroupName: 'log-group',
        retention: RetentionDays.THREE_MONTHS,
      }),
      bundling: {
        minify: true,
        keepNames: true,
        sourceMap: true,
        mainFields: ['module', 'main'],
        externalModules: ['@aws-sdk/*'],
      },
    });

    // 👇 Add permissions to read data from the table
    this.table.grantReadData(this.lambda);

    // 👇 Set the API as Lambda trigger
    this.api.root
      .resourceForPath('/object/{id}')
      .addMethod('GET', new LambdaIntegration(this.lambda));
  }
}
```

And here’s a simple code:

```ts
// lib/handler.ts
import type {
  APIGatewayProxyEventBase,
  APIGatewayProxyResult,
} from 'aws-lambda';

import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand } from '@aws-sdk/lib-dynamodb';

const dynamoDbClient = new DynamoDBClient();
const documentClient = DynamoDBDocumentClient.from(dynamoDbClient);

export const main = async (
  event: APIGatewayProxyEventBase<unknown>,
): Promise<APIGatewayProxyResult> => {
  const { pathParameters } = event;
  const id = pathParameters!.id;

  const { Item } = await documentClient.send(
    new GetCommand({ TableName: process.env.TABLE_NAME, Key: { id } }),
  );

  if (Item === undefined) {
    return { statusCode: 404, body: JSON.stringify({ message: 'Not found' }) };
  }

  return { statusCode: 200, body: JSON.stringify(Item) };
};
```

## 🚀 Step 7: Deploy and Test

```bash
cdk bootstrap
cdk deploy --all
```

Test your endpoints by accessing the API Gateway URLs from clients `api.example.com/object/some-id`.

Depending on your position, you should observe Cloudwatch Logs in one region or another!

## 🚀 Next steps:

Add authorization with Cognito.

As regional APIs, some features like API Keys will not be global.

## ✅ Benefits of This Architecture

- **Global low-latency access** to data.
- **Fully managed infrastructure** with high availability.
- **Pay-per-use** with minimal operational overhead.
- **Easily extendable** to more regions or endpoints.

## 🛠️ Conclusion

Deploying a global serverless API is simpler than ever with AWS’s managed services and the CDK. With just a few lines of code, you can serve users across the world with fast response times and consistent data. Whether you're building for mobile, IoT, or global web apps, this setup gives you performance, reliability, and scale.
