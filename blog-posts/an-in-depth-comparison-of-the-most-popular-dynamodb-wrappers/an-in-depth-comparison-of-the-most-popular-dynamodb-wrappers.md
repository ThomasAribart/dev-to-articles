---
published: false
title: An in-depth comparison of the most popular DynamoDB wrappers
cover_image: https://raw.githubusercontent.com/ThomasAribart/dev-to-articles/master/blog-posts/write-type-safe-s3-select-queries-with-kysely/type_safe_s3_select_queries_with_kysely_resized.webp
description: An in-depth comparison of the 4 most popular wrappers for the DynamoDB Client in Typescript. Which one should you chose?
tags: Typescript, DynamoDB, Wrapper, Serverless
series:
canonical_url:
---

AWS DynamoDB is a [key-value database designed to run high-performance applications at any scale](https://aws.amazon.com/dynamodb). It automatically scales up and down based on your current traffic, and does not require maintaining connections (as requests are sent over HTTP), which makes it the **go-to DB service for serverless developers on AWS**.

Because itโs 2023 and no-one writes HTTP requests anymore, AWS published a SDK called the [document client](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/dynamodb-example-dynamodb-utilities.html) to craft said requests. However, if youโve ever used it, you will know that **itโs still very painful to use**.

For instance, letโs look at this `UpdateCommand` example from the [DynamoDB docs itself](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStarted.UpdateItem.html):

```tsx
await DocumentClient.send(
  new UpdateCommand({
    TableName: 'TABLE_NAME',
    Key: {
      // ๐ No type-safety on the primary key
      title: 'MOVIE_NAME',
      year: 'MOVIE_YEAR',
    },
    // ๐ String expressions hard to build (and still no type-safety)
    UpdateExpression: 'set info.plot = :p, info.#r = :r',
    // ๐ When used in Expressions, attribute names have to be provided separately
    ExpressionAttributeNames: {
      '#r': 'rank',
    },
    // ๐คฆโโ๏ธ List of attribute names as strings separated by commas
    ProjectionExpression: '#r',
    // ๐ Attribute values have to be provided separately
    ExpressionAttributeValues: {
      // ๐ No validation or type-safety to enforce DB schema
      ':p': 'MOVIE_PLOT',
      ':r': 'MOVIE_RANK',
    },
  }),
);
```

It is a very simple example (updating two fields of a `Movie` item), yet already very verbose ๐ย?And **things only get messier as the complexity of your business logic grows**: What if your items have 20 attributes? With some of them deeply nested? Or optional? What if you want to index an item or not depending on its values (e.g. a `status` attribute)? What about polymorphism?

In those cases (which are quite common) **the required code to generate those requests can get very hard to maintain**. That is why, very early on, developers built open-source libraries to โwrapโ the DynamoDB client, with two goals in mind:

- ๐๏ธโโ๏ธย?Simplifying the writing of DynamoDB requests
- ๐ย?Adding run-time **data validation**, i.e. **โartificialโ schemas** to a schema-less DB (and more recently type-safety)

For instance, here is an example of the same `UpdateCommand` with one of those wrappers, [DynamoDB-Toolbox](https://github.com/jeremydaly/dynamodb-toolbox):

```tsx
import { Table, Entity } from 'dynamodb-toolbox';

// Provided some minor boilerplate...
const MovieTable = new Table({
  name: 'TABLE_NAME',
  partitionKey: 'title',
  sortKey: 'year',
  DocumentClient, // <= the original DocumentClient
});

const MovieEntity = new Entity({
  name: 'Customer',
  attributes: {
    title: { partitionKey: true, type: 'string' },
    year: { sortKey: true, type: 'string' },
    info: { type: 'map' },
  },
  table: MovieTable,
} as const);

// ...we get a validated AND type-safe request method ๐
await MovieEntity.update({
  title: 'MOVIE_NAME',
  year: 'MOVIE_YEAR',
  info: {
    plot: 'MOVIE_PLOT',
    rank: 'MOVIE_RANK',
  },
});
```

And just like that, we went from an obscure 18-line object to a readable and elegant 8-liner ๐คฉย?Not bad, don't you think?

DynamoDB-Toolbox is not the only SDK wrapper out there. If you browse Alex DeBrieโs [awesome-dynamodb-tools](https://github.com/alexdebrie/awesome-dynamodb#tools) repo, youโll actually find a bunch of them. So, **which one should you chose?**

In this article, we took an in depth comparison of the **4 most popular DynamoDB wrappers**:

- ๐ฆย?**Dynamoose**: Dynamoose is the OG of DynamoDB wrappers. Created in 2015, it provides a syntax that closely mirrors that of the popular Mongoose library for MongoDB.
- ๐งฐย?**DynamoDB Toolbox**: DynamoDB-Toolbox was created by Jeremy Daly, AWS Serverless Hero, and writer of the newsletter [off-by-none](https://offbynone.io/). It was first released in September 2018 and has gained in popularity ever since.
- โก๏ธ **ElectroDB**: Released in April 2020 by [Tyler W. Walsh](https://twitter.com/tinkertamper), ElectroDB benefits from having been created after Typescript won over the JS ecosystem. So it has a higher focus on type-safety than its predecessors.
- ๐ย?**DynamoDB-OneTable**: First released in January 2021, DynamoDB-OneTable is maintained by [Sensedeep](https://www.sensedeep.com/) and is part of its broader [Serverless Developer Studio](https://www.sensedeep.com/product/) offer.

We ranked them based on the following criteria:

- ๐ฃย?**Library state**: Classic open-source KPIs such as number of downloads, community, documentation etc.
- ๐๏ธย?**Data modeling**: The broadness of their Entity definition API. Do they allow attribute name re-mapping (useful for [single-table design](https://www.alexdebrie.com/posts/dynamodb-single-table/))? Do they support enums? Nested attributes definitions? Computing indexes from other attributes?
- โจย?**Typescript support**: Type-safety is all the rage these days! All libraries come with some sort of type-safety, but type-inference (i.e. inferring tailored types from custom schemas) is still hard to get right.
- ๐คย?**API**: How easy it is to do common DynamoDB requests like `put`, `get` or `query`... with secondary indexes, filters and conditions (You can find examples for each wrapper in [our dedicated repo](https://github.com/theodo/dynamodb-tools)).

## ๐ฃย?Library state

| (\*as of 2023/02) | ๐ฆย?Dynamoose | ๐งฐย?DynamoDB-Toolbox | โก๏ธ ElectroDB | ๐ย?DynamoDB-OneTable |
| --- | --- | --- | --- | --- |
| **First release date** | 2014-02-27 | 2019-11-20 | 2020-03-11 | 2021-01-12 |
| **Last release date** | โย?Jan. 6, 2023 | โย?Jan. 8, 2023 | โย?Jan. 20, 2023 | โย?Jan. 25, 2023 |
| **Github โญ๏ธ** | 1900 | 1400 | 530 | 505 |
| **NPM weekly downloads** | 86 k | 38 k | 4 k | 16 k |
| **Bundle size** | ๐กย?382 kB | โย?64.1 kB | ๐กย?176.7 kB | โย?64.3 kB |
| **Documentation** | โ | โ | ๐กย?No global search | โย?Can be improved |
| **DynamoDB Client v3 compatibility** | โ | โ | โ | โ |

All four libraries are well maintained, and have enough GitHub stars and npm downloads to be considered โbattle testedโ.

Dynamoose has the highest stats (probably from being the first one around). However, itโs the heaviest, with a size of 382KB, which is not negligible, considering that [bundles above 5MB negatively impact Lambdas cold starts](https://mikhail.io/serverless/coldstarts/aws/).

![DynamoDB Wrappers Stars History](./dynamodb-wrappers-stars-history.png)

The main takeaways are that DynamoDB-Toolbox is not compatible with the V3 of the DynamoDB client (though [it should be coming soon](https://github.com/jeremydaly/dynamodb-toolbox/pull/174)), and that the DynamoDB-OneTable documentation leaves to be desired.

## ๐๏ธย?Data modeling

We initially started with a very broad scope of features useful for Entity definition (like specifying attributes as required, or aliasing attributes). However, most of them were already implemented by all libraries. For the sake of simplicity, we removed them and kept the following ones:

- **Nested attributes definition:** Could we type nested fields of lists and maps attributes (like `plot` and `rank` in our first `Movie` example)?
- **Enum support**: Could we specify a finite range of values for a primitive attribute?
- **Default values**: Could we provide default values for an attribute? That is especially useful for entities with โsimpleโ access patterns like fixed strings. We differentiated _independent defaults_ (fixed or derived from context such as timestamps or env variables) from _dependent defaults_ (computed from other attributes).
- **Pre-save/post-fetch attribute transformation**: This can be needed for technical reasons, such as prefixing attributes. When possible, itโs best to hide such details from your code and let your wrapper handle the heavy-lifting.
- **Polymorphism support**: Sometimes, items can have different statuses and shapes that go with them. We tested how easy it was to translate to in each library.

|  | ๐ฆย?Dynamoose | ๐งฐย?DynamoDB-Toolbox | โก๏ธ ElectroDB | ๐ย?DynamoDB-OneTable |
| --- | --- | --- | --- | --- |
| **Nested attributes definition** | โ | โ | โ | โ |
| **Enum support** | โ | โ | โ | โ |
| **Independent defaults** | โ | โ | โ | โ |
| **Dependent defaults** | โ | โ | โ | ๐กย?Via string templates like `"user#${email}"` |
| **Attribute value transformation** | โ | โ | โ | โ |
| **Polymorphism** | โ | โ | โ | โ |

Overall, Dynamoose and ElectroDB have the upper hand, with ElectroDB being slightly ahead as it allows deriving attributes default values from other attributes.

Surprisingly, **none of those libraries handles polymorphism and type-safe dependent defaults**. As a maintainer of DynamoDB-Toolbox, I know for sure that those features are coming in the next major, so if youโre already using it, do not consider migrating to ElectroDB just yet ๐

## โจย?Typescript support

All libraries support Typescript at a basic level, so we mostly focused on [type inference](https://www.typescriptlang.org/docs/handbook/type-inference.html). We looked for type inference in:

- **DynamoDB requests** (root and nested level attributes)
- **Dependent defaults definition**
- **Expressions (Conditions, filters and projections)**

|  | ๐ฆย?Dynamoose | ๐งฐย?DynamoDB-Toolbox | โก๏ธ ElectroDB | ๐ย?DynamoDB-OneTable |
| --- | --- | --- | --- | --- |
| **Requests** (Root attributes) | โ | โ | โ | โ |
| **Requests** (Nested attributes) | โ | โ | โ | โ |
| **Dependent defaults** | โ | โ | โ | โ |
| **Expressions** | โ | ๐กย?yes but only at root level | โ | โย?Via string templates like `"(${role} = {admin})"` |
| **IDE performances** | โ | โย?Slow | โ | โ |

Once again ElectroDB has the upper hand here. Very nice job, Tyler W. Walsh ๐

---

## ๐คย?API

Finally we compared each solutionโs API regarding requests to DynamoDB. Warning: This one is a bit subjective. Verbosity. How natural the requests were written.

|  | ๐ฆย?Dynamoose | ๐งฐย?DynamoDB-Toolbox | โก๏ธ ElectroDB | ๐ย?DynamoDB-OneTable |
| --- | --- | --- | --- | --- |
| **Single Item Requests** | โ | ๐ก | โ | ๐ก |
| **Queries & Scans** | โ | ๐ก | โ | ๐ก |
| **Conditions** | โ | ๐ก Not intuitive | โ | โ |
| **Filters** | โ | ๐ก | โ | โ |

ElectroDB has a better API for querying, name your indexes with business sense

```tsx
const movies = await Movie.query.byType({ type: 'horror' }).go();
```

`Dynamodb-toolbox`

```tsx
const { Item } = await PokemonInstanceEntity.get(pokemonMasterId, {
  index: 'GSI', // <= Technical index name, really clear
});
```

## Conclusion

Overall, as of march 2023, **ElectroDB looks like the best DynamoDB client wrapper**. Although it is newer and less โbattle testedโ, it is better than its concurrent on every other criteria: It has the **same data modeling features**, a **more complete type inference**, and a **nicer API**.

That being said, there are some parts to improve:

- Type inference still has some blind spots
- There's no support for polymorphism
- Entity definition autocompletion could be more helpful (it would benefit from a [zod-like approach](https://github.com/colinhacks/zod))

Also, Iโm not a fan of the `find` and `match` methods it exposes, which are not native DynamoDB requests and can build costly and inefficient `scan` requests without you being aware. Otherwise, it is a good match!

Finally, **I would not rule out DynamoDB-Toolbox just yet!** Its next major is just around the corner, with many new capabilities that even ElectroDB doesnโt have (such as type-safe dependent defaults and polymorphism). So expect a round 2 of this article in the next few monthsโฆ ๐
