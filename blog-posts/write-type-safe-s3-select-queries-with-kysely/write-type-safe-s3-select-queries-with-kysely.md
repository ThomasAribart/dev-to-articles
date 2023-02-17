---
published: false
title: 'Write type-safe S3 Select queries with Kysely'
cover_image: ./type_safe_s3_select_queries_with_kysely.webp
description: 'Write type-safe S3 Select queries with Kysely'
tags: Typescript, S3, SQL, Serverless
series:
canonical_url:
---

S3 Select is an Amazon S3 feature that enables [retrieving subsets of S3 Objects content via SQL expressions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-glacier-select-sql-reference-select.html). You can use clauses like SELECT and WHERE to fetch data from CSVs, JSONs, or Apache Parquet files, even if they are compressed with GZIP and/or server-side encrypted.

S3 Select is simple to use, [cost-effective](https://aws.amazon.com/s3/pricing/) and can [drastically improve the performances of your application](https://aws.amazon.com/fr/blogs/storage/run-queries-up-to-9x-faster-using-trino-with-amazon-s3-select-on-amazon-emr/) depending on your query. It is overall a great addition to the Serverless developer toolkit, particularly when:

- You need to access a large amount of data (which would make it unfit for [other storage solutions like DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/ServiceQuotas.html#limits-items))
- You need to query and filter it in a complex and/or dynamic way
- You don’t need to use a JOIN clause (as S3 Select can only query a single file) or to update a single record of the data (it is called S3 Select, not S3 Insert 🙂)

In this article, we’ll learn how to run S3 Select commands in Typescript, and how we can improve our DevX and type-safety using [Kysely](https://github.com/koskimas/kysely).

## Querying with S3 Select

Let’s say that we have a DB of Pokemons in the shape of a CSV, stored somewhere in a S3 Bucket:

```csv
id ; name      ; customName ; type     ; level ; generation
1  ; pikachu   ;            ; electric ; 42    ; 1
2  ; charizard ;            ; fire     ; 54    ; 1
3  ; meganium  ; plantyDino ; grass    ; 26    ; 2
...
```

What if we want to retrieve the fire Pokemons from generation 1 and 2? Well, S3 Select let us do that with the following query:

```tsx
import { S3Client, SelectObjectContentCommand } from '@aws-sdk/client-s3';

export const s3Client = new S3Client({
  region: 'us-east-1', // <= Your region here
});

const { Payload } = await s3Client.send(
  new SelectObjectContentCommand({
    // 👇 Those first params are required but don't mind them
    ExpressionType: 'SQL',
    OutputSerialization: {
      JSON: {
        RecordDelimiter: ',',
      },
    },
    // 👇 Those depends on the CSV
    InputSerialization: {
      CSV: {
        FileHeaderInfo: 'USE',
        FieldDelimiter: ';',
        QuoteCharacter: '"',
      },
    },
    // 👇 Those are the most important
    Bucket: 'my-super-bucket-name',
    Key: 'pokedex.csv',
    Expression: `
			select "id", "name", "customName", "type" as "pokemonType", "level"
				from "S3Object"
				where
					"generation" in ('1', '2')
					and "type" = 'fire'
	  `,
  }),
);

// Note that this command requires the s3:GetObject permission
```

This is great and all but we can make it better:

- 📝 We need to add some custom parsing to actually use the response `Payload`
- 👍 We can use Kysely to produce the SQL expression in a type-safe and devX-friendly way
- 🌈 We can also use it to type the command result

## Parsing the query response

The `Payload` is not a plain Javascript object but an instance of `AsyncInterable`. Now what the hell is an `AsyncIterable` you ask me? Well, you could look for the official [Async Iterator documentation on MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#the_async_iterator_and_async_iterable_protocols)… or you can just stop asking questions and use the following helper:

```tsx
import type { SelectObjectContentEventStream } from '@aws-sdk/client-s3';

// 👇 TextDecoder is a native Node/Browser class
const textDecoder = new TextDecoder();

const parseS3SelectEventStream = async (
  s3SelectEventStream:
    | AsyncIterable<SelectObjectContentEventStream>
    | undefined,
): Promise<unknown[]> => {
  if (!s3SelectEventStream) {
    return [];
  }

  const stringifiedJSONOutputs: string[] = [];

  for await (const event of s3SelectEventStream) {
    if (event.Records) {
      const stringifiedJSONOutput = textDecoder.decode(event.Records.Payload);
      stringifiedJSONOutputs.push(stringifiedJSONOutput);
    }
  }

  const rows = JSON.parse(
    '[' + stringifiedJSONOutputs.join('').slice(0, -1) + ']',
  ) as unknown[];

  return rows;
};
```

Now we can parse the output like this:

```tsx
const { Payload: s3SelectEventStream } = await s3Client.send(
  new SelectObjectContentCommand({
    // ...
  }),
);

// 🎉 Ta-da!
const myPokemons: unknown[] = await parseS3SelectEventStream(
  s3SelectEventStream,
);
```

## Building the query with Kysely

Let’s face it, writing SQL queries by hand is a pain and very error prone. Besides, wouldn't it be nice to have some type error if the CSV suddenly changes shape?

That’s where Kysely comes to the rescue: Kysely is a [type-safe and devX-friendly typescript SQL query builder](https://github.com/koskimas/kysely). It was designed to work with PostgreSQL and MySQL, but it exposes a few classes that can let us write queries without being connected to an actual relational database.

Let’s begin by designing our CSV type:

```tsx
enum PokemonType {
  Water = 'water',
  Grass = 'grass',
  Fire = 'fire',
  // ...
}

interface PokemonCSV {
  id: string;
  name: string;
  customName?: string;
  type: PokemonType;
  // 👇 In CSVs everything is a string
  level: string;
  generation: '1' | '2'; // ...up to 9
}
```

Next, let’s install Kysely and instanciate a `Kysely` database:

```bash
# npm
npm install kysely

# yarn
yarn add kysely
```

```tsx
import {
  Kysely,
  DummyDriver,
  SqliteAdapter,
  SqliteIntrospector,
  SqliteQueryCompiler,
} from 'kysely';

interface Database {
  S3Object: PokemonCSV;
}

const db = new Kysely<Database>({
  dialect: {
    createAdapter: () => new SqliteAdapter(),
    createDriver: () => new DummyDriver(),
    createIntrospector: ($db: Kysely<unknown>) => new SqliteIntrospector($db),
    createQueryCompiler: () => new SqliteQueryCompiler(),
  },
});

// That’s it 🎉
```

Simple, no? Now, let’s write the same query, but while enjoying some type-safety and auto-completion:

```tsx
const kyselyQuery = db
  .selectFrom('S3Object')
  .select([
    'id',
    'name',
    'customName',
    // 🙌 You can rename columns as you like
    'type as pokemonType',
    'level',
    // 💥 Will trigger an error:
    'unexistingColumn',
  ])
  // 🙌 Every method is type-safe!
  .where('generation', 'in', ['1', '2'])
  .where('type', '=', PokemonType.Fire);
```

To protect us from nasty SQL injections, Kysely doesn’t directly provide us with the SQL expression but with a `sql` string and `parameters` array:

```tsx
const {
  sql, // 👈 SQL query with '?' as placeholders
  parameters, // 👈 Array of parameters
} = kyselyQuery.compile();
```

Because S3 Select doesn’t accept parameters in its API, we have to hydrate the parameters ourselves:

```tsx
const dangerouslyHydrateSQLParameters = (
  sql: string,
  parameters: readonly unknown[],
): string => {
  for (const parameter of parameters) {
    sql = sql.replace('?', `'${String(parameter)}'`);
  }

  return sql;
};

const { sql, parameters } = kyselyQuery.compile();

// ⛔️ **BE SURE TO VALIDATE DYNAMIC PARAMETERS FIRST** ⛔️
const sqlExpression = dangerouslyHydrateSQLParameters(sql, parameters);
console.log(sqlExpression);

// 👇 We retrieve the same expression as above:
// select "id", "name", "customName", "type" as "pokemonType", "level"
//   from "S3Object"
//   where
//     "generation" in ('1', '2')
//     and "type" = 'fire'
```

We just have to provide it to our S3 Select command and voilà! We’re done!

Well almost: Notice that we only typed our SQL query expression! What about the response that comes from S3?

## Inferring the response type

Initially, Kysely was not only made to build queries but also execute them. We can just benefit from the inferred type by inspecting the `execute` property of our Kysely query. That can be done with the help of some TS wizardry:

```tsx
import type { SelectQueryBuilder } from 'kysely';

type QueryResponseRow<
  // 👇 Add a large type constraint
  KyselyQuery extends SelectQueryBuilder<
    Record<string, unknown>,
    string,
    unknown
  >,
> =
  // 👇 Remove the Promise wrapper
  Awaited<
    // 👇 Get the return type of the execute method
    ReturnType<KyselyQuery['execute']>
    // 👇 Unpack the array
  >[number];

type Pokemon = QueryResponseRow<typeof kyselyQuery>;

// 👇 Equivalent to:
type Pokemon = {
  id: string;
  name: string;
  // 👍 customName is indeed possibly undefined
  customName: string | undefined;
  level: string;
  // 🙌 "type" property has been renamed
  pokemonType: PokemonType;
};
```

Now we can just use this type the response of our `parseS3SelectEventStream` util and voilà! We’re done! For good this time 🙂

## Conclusion

Both S3 Select and Kysely are awesome tools. By joining both of them, we can run performant, scalable and cost-effective queries while benefitting from strong and DRY type-safety and inference 🥳

Note that there is one minor drawback, though: Kysely will add 120KB in your Lambdas bundles (props to for [serverless-analyze-bundle-plugin](https://github.com/adriencaccia/serverless-analyze-bundle-plugin/) for helping me out with this 🙌). It is not a lot, but not negligible either as [NodeJS Lambdas bundles above 5MB negatively impacts their cold starts](https://mikhail.io/serverless/coldstarts/aws/). So you might want to re-evaluate adding Kysely to your bundles if your query is not changing often.
