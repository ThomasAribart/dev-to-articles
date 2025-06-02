---
published: false
title: 'MCP Client: Building a Smart and Robust Integration to DynamoDB with DynamoDB-Toolbox'
cover_image: TODO
description: Leverage schema-aware logic and defined access patterns for effortless integration
tags: AI, MCP Server, Typescript, Serverless, AWS, DynamoDB
---

MCP Clients are gaining popularity and making backend interactions more intuitive and powerful.

However, connecting them to a database like [DynamoDB](https://aws.amazon.com/dynamodb/) can be tricky. DynamoDB’s schema-less design and strict dependence on well-thought-out access patterns mean one misstep can cause your application to fall apart.

Enter [DynamoDB-Toolbox](https://www.dynamodbtoolbox.com/) — A lightweight and type-safe query builder for DynamoDB that just released a tooling to deploy

- Smart: Only efficient queries, fast and cost-efficient
- Robust: Cannot insert invalid data
- DRY: One line of code

Let’s build a real-world example using DynamoDB-Toolbox and leverage it to power an MCP Server.

---

## 1. Preparing our Models

This part has not much to do with LLMs, but rather with your Models definition. If you already use DynamoDB-Toolbox you can skip to the next section. Otherwise, this is a nice introduction to how the Toolbox works :)

Let’s model a **Pokedex** with two entities — `Trainer` and `Pokemon` — with the following requirements

- Query all trainers efficiently, ordered by `birthDate`.
- List all Pokemons of a specific trainer, ordered by `captureDate`.

We’ll use a **Single Table Design**, avoiding a sort key in the primary index but using one in a dedicated GSI.

### Create a Table

First, let's **create a Table** in AWS. Here's an example using the [AWS CDK](https://aws.amazon.com/cdk/):

```ts
import { AttributeType, TableV2 } from 'aws-cdk-lib/aws-dynamodb';

new TableV2(app, 'PokeTable', {
  tableName: 'poke-table',
  partitionKey: { name: 'PK', type: AttributeType.STRING },
  globalSecondaryIndexes: [
    {
      indexName: 'GSI1',
      partitionKey: { name: 'GSI1_PK', type: AttributeType.STRING },
      sortKey: { name: 'GSI1_SK', type: AttributeType.STRING },
    },
  ],
});
```

### Create a `Table` Instance

```ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { Table } from 'dynamodb-toolbox/table';

const documentClient = DynamoDBDocumentClient.from(new DynamoDBClient());

const PokeTable = new Table({
  name: 'poke-table',
  partitionKey: { name: 'PK', type: 'string' },
  indexes: {
    GSI1: {
      type: 'global',
      partitionKey: { name: 'GSI1_PK', type: 'string' },
      sortKey: { name: 'GSI1_SK', type: 'string' },
    },
  },
  meta: {
    title: 'Pokedex',
    description: 'Single Table containing Trainers and their Pokemons',
  },
  documentClient,
});
```

### Define Schemas and Entities

Here's an example for the `Trainer` Entity:

```ts
import { Entity } from 'dynamodb-toolbox/entity';
import { item, string } from 'dynamodb-toolbox/schema';

const TrainerEntity = new Entity({
  name: 'trainer',
  table: PokeTable,
  schema: item({
    trainerId: string().key().savedAs('PK'),
    name: string(),
    birthDate: string(),
    bio: string().optional(),
    profilePictureUrl: string(),
  }).and(prevSchema => ({
    GSI1_PK: string().default('trainers').hidden(),
    GSI1_SK: string()
      .link<typeof prevSchema>(({ birthDate }) => birthDate)
      .hidden(),
  })),
  meta: {
    title: 'Trainer',
    description: 'Represents a Trainer',
  },
});
```

And one for the `Pokemon` Entity:

```ts
import { Entity } from 'dynamodb-toolbox/entity';
import { item, string, number, boolean, list } from 'dynamodb-toolbox/schema';

const PokemonEntity = new Entity({
  name: 'pokemon',
  table: PokeTable,
  schema: item({
    pokemonId: string().key().savedAs('PK'),
    species: string(),
    pokeTypes: list(string()),
    level: number(),
    legendary: boolean().optional(),
    trainerId: string().optional(),
    captureDate: string().optional(),
  }).and(prevSchema => ({
    GSI1_PK: string()
      .link<typeof prevSchema>(
        ({ trainerId }) => `trainers/${trainerId}/pokemons`,
      )
      .hidden(),
    GSI1_SK: string()
      .link<typeof prevSchema>(({ captureDate }) => captureDate)
      .hidden(),
  })),
  meta: {
    title: 'Pokemon',
    description: 'Represents a Pokemon. Can be wild or captured.',
  },
});
```

### Create Access Patterns

Let's define two [Access Patterns](https://www.dynamodbtoolbox.com/docs/entities/actions/access-pattern) that follows our requirements:

```ts
import { any } from 'dynamodb-toolbox/schema';
import { AccessPattern } from 'dynamodb-toolbox/entity/actions/accessPattern';

const trainersAccessPattern = TrainerEntity.build(AccessPattern)
  // 👇 Does not require any input
  .schema(any())
  .pattern(() => ({ index: 'GSI1', partition: 'trainers' }))
  .meta({
    title: 'Query all trainers',
    description: 'Returns all trainers (paginated)',
  });

const trainerPokemonsAccessPattern = PokemonEntity.build(AccessPattern)
  .schema(string())
  .pattern(trainerId => ({
    index: 'GSI1',
    partition: `trainers/${trainerId}/pokemons`,
  }))
  .meta({
    title: 'Trainer’s Pokemons',
    description:
      'Returns all Pokemons for a given trainer, ordered by capture date (paginated)',
  });
```

---

## 2. Launch Your MCP Server

Now that we have our `Table`, `Entities` and `AccessPatterns` ready, we're almost ready to launch our MCP Server!

### Define a `Database`

Let's bundle them in a [`Registry`](https://www.dynamodbtoolbox.com/docs/tables/actions/registry) and [`Database`](https://www.dynamodbtoolbox.com/docs/databases/usage) features to serve as a single entry point for our data:

```ts
import { Registry } from 'dynamodb-toolbox/table/actions/registry';
import { Database } from 'dynamodb-toolbox/database';

const PokeTableRegistry = PokeTable.build(Registry)
  .registerEntities(TrainerEntity, PokemonEntity)
  .registerAccessPatterns({
    trainers: trainersAccessPattern,
    trainerPokemons: trainerPokemonsAccessPattern,
  });

const PokeDB = new Database({
  pokedex: PokeTableRegistry,
  // You can add more Table registries here
});

// Ex: Get an Entity by `name`
PokeDB.tables.pokedex.entities.pokemon
// Ex: Get an AccessPattern by key
PokeDB.tables.pokedex.accessPatterns.trainers.query(...)
```

### Authenticate to AWS

For development, we’ll run a [local MCP server](https://modelcontextprotocol.io/quickstart/server), but you could also deploy this MPC over HTTP, for instance using Frederic Barthelet's awesome [Middy MCP](https://github.com/fredericbarthelet/middy-mcp) middleware.

In our case, we can authenticate to AWS using our [SSO profile](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-credential-providers/):

```ts
import { fromSSO } from '@aws-sdk/credential-providers';

const dynamoDbClient = new DynamoDBClient({
  region: '<AWS_REGION>',
  profile: '<SSO_PROFILE>',
  credentials: fromSSO(),
});
```

### Launch the `McpServer`

Now let's start our [`McpServer`](https://modelcontextprotocol.io/quickstart/server) and use the [`MCPToolkit`](https://www.dynamodbtoolbox.com/docs/databases/actions/mcp-toolkit) from DynamoDB-TOolbox to quickly wire your DynamoDB access:

```ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const mcpServer = new McpServer({
  name: 'poke-server',
  version: '1.0.0',
});

// 👇 That's it!
PokeDB.build(MCPToolkit).addTools(mcpServer);

async function main() {
  const transport = new StdioServerTransport();
  await mcpServer.connect(transport);
  console.log('Poke-MCP Server running on stdio');
}

main().catch(error => {
  console.error('Fatal error:', error);
  process.exit(1);
});
```

Now we can start interacting with our DynamoDB Tables using natural language!

---

## 3. Run Your MCP Client 🙌

I used Claude as the client. Simply add your new Tool in the settings and restart the client.

_👉 **TODO**: Add a screenshot here._

You’ll be able to query trainers, list Pokemons, and insert data interactively.

_👉 **TODO**: Add a screenshot here._

We see that Claude has been able to connect to our DynamoDB 🙌 Of course there are no Trainers yet but we can tell it to add one with some starter pokemons:

_👉 **TODO**: Add a screenshot here._

---

## Conclusion

With DynamoDB-Toolbox and the MCP ecosystem, building smart and maintainable integrations to DynamoDB is easier than ever.

- Smart: Only efficient queries, fast and cost-efficient
- Robust: Cannot insert invalid data
- DRY: One line of code

Now go catch ‘em all — and let your AI Assistant help! 🧠🎮
