---
published: false
title: The DynamoDB-Toolbox v1 beta is here 🙌 All you need to know!
cover_image: https://raw.githubusercontent.com/ThomasAribart/dev-to-articles/master/blog-posts/an-in-depth-comparison-of-the-most-popular-dynamodb-wrappers/an-in-depth-comparison-of-the-most-popular-dynamodb-wrappers-2.png
description: The DynamoDB-Toolbox v1 beta is here 🙌 All you need to know!
tags: Typescript, DynamoDB, AWS, Serverless
series:
canonical_url:
---

At [Kumo](https://dev.to/kumo), we are a big fan of Jeremy Daly’s [DynamoDB-Toolbox](https://github.com/jeremydaly/dynamodb-toolbox). We started using it as early as 2019. We grew fond of it... but were also too well aware of its flaws 😅

One of them was that it was originally coded in JavaScript. Although Jeremy’s migrated the source code to TypeScript in 2020, it was limited to the base interface. It didn't handle type inference, a feature that I came to implement myself in the [v0.4]().

However, there were still some other features that we felt clearly lacked: From something as simple as declaring `enums` on primitive values, to having deeply nested typings (lists and maps sub-attributes) and polymorphism.

I also disliked the object-oriented approach: I don’t have anything against classes, but they are not tree-shakable, so they should be kept relatively small in the Serverless world. That’s what AWS went for with the [v3 of their SDK](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-dynamodb/#usage), and for good reasons: Keep bundle tights! That wasn't the case for DynamoDB-Toolbox: I remember working on an `.update` method that was more than 1000 lines long... Why bundle it when you don't even need it?

So last year, I decided to throw myself into a complete overhaul of the code, with three main objectives:

- Support v3 of the AWS SDK (although it [has been added in v0.8](https://github.com/jeremydaly/dynamodb-toolbox#using-aws-sdk-v2))
- Get it the API and type inference on par with more modern tools like [zod](https://github.com/colinhacks/zod) and [electrodb](https://electrodb.fun/)
- Bring a more functional and tree-shakable approach

Today, I am happy to announce the **v1 beta of dynamodb-toolbox is out** 🙌 It includes new `Table` and `Entity` classes, as well as complete support for `PUT`, `GET` and `DELETE` commands (including conditions and projections). With `UPDATE`, `QUERY` and `SCAN` commands soon to follow.

This article will guide you as to how the new API works, as well as the main breaking changes since the pre-v1 version - Which, by the way, only concerns the API: If you already use the `v0.x` in production, you won’t have to worry about any data migration 🥳

## Table of content

- [Installation](#installation)
- [Tables](#tables)
- [Entities](#entities)
  - [Timestamps](#timestamps)
  - [Matching the Table schema](#matching-the-table-schema)
  - [`SavedItem` and `FormattedItem`](#saveditem-and-formatteditem)
- [Designing Entity schemas](#designing-entity-schemas)
  - [Schema definition](#schema-definition)
  - [Attributes types](#attributes-types)
    - [`any`](#any)
    - [`<primitive>`](#primitives)
    - [`set`](#set)
    - [`list`](#list)
    - [`map`](#map)
    - [`record`](#record)
    - [`anyOf`](#anyof)
    - [Looking forward](#looking-forward)
- [Computed defaults](#computed-defaults)
- [Commands](#commands)
  - [`PutItemCommand`](#putitemcommand)
  - [`GetItemCommand`](#getitemcommand)
  - [`DeleteItemCommand`](#deleteitemcommand)
- [Utility helpers and types](#utility-helpers-and-types)
  - [`formatSavedItem`](#formatsaveditem)
  - [`Condition` and `parseCondition`](#condition-and-parsecondition)
  - [`Projection` and `parseProjection`](#projection-and-parseprojection)
  - [`KeyInput` and `PrimaryKey`](#keyinput-and-primarykey)
- [Errors](#errors)
- [Conclusion](#conclusion)

## Installation

```bash
### npm
npm i dynamodb-toolbox@1.0.0-beta.0

# yarn
yarn add dynamodb-toolbox@1.0.0-beta.0

## ...and so on
```

<aside style="font-size: medium;">
☝️ *Stay up to date with the patches by following the project [GitHub releases](https://github.com/jeremydaly/dynamodb-toolbox/releases)*

</aside>

The `v1` is built on top the `v3` of the AWS SDK. It has `@aws-sdk/client-dynamodb` and `@aws-sdk/lib-dynamodb` as peer dependencies so you’ll have to install them as well:

```bash
## npm
npm i @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb

## yarn
yarn add @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb

## ...and so on
```

## Tables

Tables are defined pretty much the same way is in previous versions, but the key attributes now have a type along with their name:

```tsx
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
// Will be renamed Table in the official release 😉
import { TableV2 } from 'dynamodb-toolbox';

const dynamoDBClient = new DynamoDBClient({});
const documentClient = DynamoDBDocumentClient.from(dynamoDBClient);

const MyTable = new TableV2({
  name: 'MySuperTable',
  partitionKey: {
    name: 'PK',
    type: 'string', // 'string' | 'number' | 'binary'
  },
  sortKey: {
    name: 'SK',
    type: 'number',
  },
  documentClient,
});
```

<aside style="font-size: medium;">
☝️ *The v1 does not support indexes yet as queries are not yet available.*

</aside>

The table name can be provided with a getter. This can useful in environments in which it is not actually available (such as tests or deployments):

```tsx
const MyTable = new TableV2({
  ...
  // 👇 Will only be executed at command execution time
  name: () => process.env.TABLE_NAME,
});
```

As in previous versions, the `v1` classes tag your data with an entity identifier through an internal `entity` string attribute, saved as `"_et"` by default. This can be renamed at the `Table` level through the `entityAttributeSavedAs`:

```tsx
const MyTable = new TableV2({
  ...
  // 👇 defaults to "_et"
  entityAttributeSavedAs: '__entity__',
});
```

## Entities

For Entities, the main change is that the `attributes` argument becomes `schema`:

```tsx
// Will be renamed Entity in the official release 😉
import { EntityV2, schema } from "dynamodb-toolbox"

const myEntity = new EntityV2({
  name: "MyEntity",
  table: myTable,
  // Attribute definition
  schema: schema({ ... })
})
```

### Timestamps

The internal timestamp attributes are also there and behave similarly as the [previous versions](https://www.dynamodbtoolbox.com/docs/entity#specifying-entity-definitions). You can set the `timestamps` to `false` to disable them (default value is `true`), or fine-tune the `created` and `modified` attributes names and aliases:

```tsx
const myEntity = new EntityV2({
  ...
  // 👇 de-activate timestamps altogether
  timestamps: false,
});

const myEntity = new EntityV2({
  ...
  timestamps: {
    // 👇 de-activate only `created` attribute
    created: false,
    modified: true,
  },
});

const myEntity = new EntityV2({
  ...
  timestamps: {
    created: {
      // 👇 defaults to "created"
      name: 'creationDate',
      // 👇 defaults to "_ct"
      savedAs: '__createdAt__',
    },
    modified: {
      // 👇 defaults to "modified"
      name: 'lastModificationDate',
      // 👇 defaults to "_md"
      savedAs: '__lastMod__',
    },
  },
});
```

### Matching the Table schema

An important change from previous versions is that the `EntityV2` schema is validated against the `TableV2`, both in types and at runtime. There are two ways of matching the table schema:

- The simplest one is to have an entity schema that **already matches the table schema** (see ["Designing Entity Schemas"](#designing-entity-schemas)). The Entity is then considered valid and no other argument is required:

```tsx
const pokemonEntity = new EntityV2({
  name: 'Pokemon',
  table: MyTable, // <= { PK: string, SK: string } primary key
  schema: schema({
    // Provide a schema that matches the primary key
    PK: string().key(),
    // 🙌 using "savedAs" will also work
    pokemonId: string().key().savedAs('SK'),
    ...
  }),
});
```

- If the entity schema doesn’t match the table schema, the `Entity` class will require you to add a `computeKey` property which must derive the primary key from the schema `key` attributes:

```tsx
const pokemonEntity = new EntityV2({
  // ...
  table: MyTable, // <= { PK: string, SK: string } primary key
  schema: schema({
    pokemonClass: string().key(),
    pokemonId: string().key(),
    ...
  }),
  // 🙌 `computeKey` is correctly typed
  computeKey: ({ pokemonClass, pokemonId }) => ({
    PK: pokemonClass,
    SK: pokemonId,
  }),
});
```

### SavedItem and FormattedItem

If you feel lost, you can always use the `SavedItem` and `FormattedItem` utility type to infer the type of your entity items:

```tsx
import type { FormattedItem, SavedItem } from 'dynamodb-toolbox';

const pokemonEntity = new EntityV2({
  name: 'Pokemon',
  timestamps: true,
  table: MyTable,
  schema: schema({
    pokemonClass: string().key().savedAs('PK'),
    pokemonId: string().key().savedAs('SK'),
    level: number().default(1),
    customName: string().optional(),
    internalField: string().hidden(),
  }),
});

// What Pokemons will look like in DynamoDB
type SavedPokemon = SavedItem<typeof pokemonEntity>;
// 🙌 Equivalent to:
//	{
//		_et: "Pokemon",
//		_ct: string,
//		_md: string,
//		PK: string,
//		SK: string,
//		level: number,
//		customName?: string | undefined,
//		internalField: string | undefined,
//	}

// What fetched Pokemons will look like in your code
type FormattedPokemon = FormattedItem<typeof pokemonEntity>;
// 🙌 Equivalent to:
//	{
//		created: string,
//		modified: string,
//		pokemonClass: string,
//		pokemonId: string,
//		level: number,
//		customName?: string | undefined,
//	}
```

## Designing Entity schemas

Now let’s dive into the part that received the most significant overhaul: **Schema definition**.

### Schema definition

Similarly to [zod](https://github.com/colinhacks/zod) or [yup](https://github.com/jquense/yup), attributes are now defined through function builders. For TS users, this removes the need for the `as const` statement previously needed for type inference (so don't forget to remove it when you migrate! 🙈).

You can either import the attribute builders through their dedicated imports, or through the `attribute` or `attr` shorthands. For instance, those declarations will output the same attribute schema:

```tsx
import { string, attribute, attr } from 'dynamodb-toolbox';

// 👇 More tree-shakable
const pokemonName = string();
// 👇 Not tree-shakable, but single import
const pokemonName = attribute.string();
const pokemonName = attr.string();
```

Prior to being wrapped in a `schema` declaration, attributes are called **warm:** They are **not validated** (at run-time) and can be used to build other schemas. By inspecting their types, you will see that they are prefixed with `$`. Once **frozen**, validation is applied and building methods are stripped:

_TODO GIF OF SCREEN CAPTURE_

The main takeaway is that **warm schemas can be composed** while **frozen schemas cannot**:

```tsx
import { schema } from 'dynamodb-toolbox';

const pokemonName = string();

const pokemonSchema = schema({
  // 👍 No problem
  pokemonName,
  ...
});

const pokedexSchema = schema({
  // ❌ Not possible
  pokemon: pokemonSchema,
  ...
});
```

You can create/update warm attributes by using dedicated methods or by providing an option object. The first method provides a **slick devX** with autocomplete and shorthands, while the second one theoretically requires **less compute time and memory usage**, although it should be very minor (validation being only applied on freeze):

```tsx
// Using methods
const pokemonName = string().required('always');
// Using options
const pokemonName = string({ required: 'always' });
```

All attributes share the following options:

- `required`: Tag a root attribute or Map sub-attribute as **required**. Possible values are:
  - `"atLeastOnce"` _(default)_ Required in `PUT`s
  - `"never"`: Optional in `PUT`s
  - `"always"`: Required in `PUT`s and `GET`s

```tsx
// Equivalent
const pokemonName = string().required();
const pokemonName = string({ required: 'atLeastOnce' });

// `.optional()` is a shorthand for `.required(”never”)`
const pokemonName = string().optional();
const pokemonName = string({ required: 'never' });
```

A very important breaking change from previous versions is that **root attributes and Map sub-attributes are now required by default**. This was made so **composition and validation work better together**.

<aside>
💡 *Outside of root attributes and Map sub-attributes, such as in a list of strings, it doesn’t make sense for sub-schemas to be optional. So, should string validation and type inference depend on the context (ignore `required` in Lists but not in Maps) OR force users to write `list(string().required())` every time? It felt more straightforward to enforce `string()` as required by default and prevent schemas such as `list(string().optional())`.*

</aside>

- `hidden`: Skip attribute when formatting the returned item of a command.

```tsx
const pokemonName = string().hidden();
const pokemonName = string({ hidden: true });
```

- `key`: Tag attribute as needed for computing the item primary key.

```tsx
// Note: The method will also modify the `required` property to "always"
// (it is often the case in practice, you can still use `.optional()` if needed)
const pokemonName = string().key();
const pokemonName = string({ key: true });
```

- `savedAs`: (previously known as `map`) Rename a root or Map sub-attribute before sending write commands.

```tsx
const pokemonName = string().savedAs('_n');
const pokemonName = string({ savedAs: '_n' });
```

- `default`: _[TO FINISH]_ Most attribute types expose a `default` option… Although only primitives … Let me know if you need it.

### Attributes types

Here’s the list of available attribute types:

#### Any

Define an attribute of any value. No validation will be applied at runtime, and its type will be resolved as `unknown`.

```tsx
import { any } from 'dynamodb-toolbox';

const pokemonSchema = schema({
  // ...
  metadata: any(),
});

type FormattedPokemon = FormattedItem<typeof pokemonEntity>;
// => {
//		...
//		metadata: unknown
//	}
```

#### Primitives

Define a `string`, `number`, `boolean` or `binary` attribute.

```tsx
import { string, number, boolean, binary } from "dynamodb-toolbox"

const pokemonSchema: schema({
	// ...
	pokemonType: string(),
	level: number(),
	isLegendary: boolean(),
	binEncoded: binary(),
})

type FormattedPokemon = FormattedItem<typeof pokemonEntity>
// => {
//		...
//		pokemonType: string
//		level: number
//		isLegendary: boolean
//		binEncoded: Buffer
//	}
```

Primitive types have an additional `enum` option. For instance, you could define the `pokemonType`:

```tsx
import { string } from 'dynamodb-toolbox';

const pokemonTypeAttribute = string().enum('fire', 'grass', 'water');

// `.const("POKEMON")` is a shorthand for `.enum("POKEMON").default("POKEMON")`
const pokemonPartitionKey = string().const('POKEMON');
```

<aside>
💡 *For type inference reasons the `enum` options in primitives is only available as method, not as an option object

</aside>

#### Set

Defines a set of strings, numbers or binaries:

```tsx
import { set } from 'dynamodb-toolbox';

const pokemonSchema = schema({
  // ...
  skills: set(string()),
});

type FormattedPokemon = FormattedItem<typeof pokemonEntity>;
// => {
//		...
//		skills: Set<string>
//	}
```

Unlike in the previous versions, sets are kept as `Set` classes. Let me know if you would rather use `Array` or to be able to chose from both.

#### List

Defines a list of sub-schemas of any type:

```tsx
import { list } from 'dynamodb-toolbox';

const pokemonSchema = schema({
  // ...
  skills: list(string()),
});

type FormattedPokemon = FormattedItem<typeof pokemonEntity>;
// => {
//		...
//		skills: string[]
//	}
```

#### Map

Defines a finite list key-value pairs. As for Lists, Map attributes are (finally!) recursive without any limitation of level. Leverage of recursive typing:

```tsx
import { map } from 'dynamodb-toolbox';

const pokemonSchema = schema({
  nestedMagic: map({
    will: map({
      work: string().const('!'),
    }),
  }),
});

type FormattedPokemon = FormattedItem<typeof pokemonEntity>;
// => {
//		...
//		nestedMagic: {
//			will: {
//				work: "!"
//			}
//		}
//	}
```

#### Record

A new attribute type that translates to `Partial<Record<KeyType, ValueType>>` in TypeScript. _[TO FINISH]_ Partial, accept an indefinite number of properties:

```tsx
const .. = record(string().enum("foo", "bar"), number())
```

#### AnyOf

A new **meta-**attribute type that represents a union of types, i.e. a range of possible types:

```tsx
import { anyOf } from 'dynamodb-toolbox';

const pokemonSchema = schema({
  // ...
  pokemonType: anyOf([
    string().const('fire'),
    string().const('grass'),
    string().const('water'),
  ]),
});
```

In this particular case, an `enum` would have done the trick. However, `anyOf` becomes particularly powerful when used in conjunction with `map` and `enum` or `const` directives of a primitive attribute, to implement **polymorphism**:

```tsx
const pokemonSchema = schema({
  // ...
  captureState: anyOf([
    map({
      status: string().const('catched'),
      // 👇 captureState.trainerId exists if status is "catched"...
      trainerId: string(),
    }),
    // ...but not otherwise! 🙌
    map({ status: string().const('wild') }),
  ]),
});

type CaptureState = FormattedItem<typeof pokemonEntity>['captureState'];
// 🙌 Equivalent to:
//  | { status: "wild" }
//  | { status: "catched", trainerId: string }
```

#### Looking forward

That’s all for now! I’m planning to include new `tuple` and `allOf` attributes someday. Let me know if this would be a feature you’d need!

## Computed defaults

_[TO FINISH]_ The `default` only support independent defaults. In previous versions, `default` could also compute. This feature was very handy … for technical … such as computing a GSI from other attributes.

However, it was just impossible to type correctly in TypeScript:

```tsx
const pokemonSchema = schema({
	// ...
	level: number(),
	levelPlusOne: number().default(
		// ❌ No way to retrieve the caller context
		input => input.level + 1
})
```

It means the `input` was typed as any and it fell to the developper to type it correctly, which just didn’t cut it for me.

The solution I committed to was to split dependent defaults declaration into 2 steps:

- First, **declare that an attribute default should be derived from other attributes**:

```tsx
import { ComputedDefault } from 'dynamodb-toolbox';

const pokemonSchema = schema({
  // ...
  level: number(),
  levelPlusOne: number().default(ComputedDefault),
});
```

<aside>
💡 `ComputedDefault` is a JavaScript [Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) (*TLDR: A sort of unique and custom `null`*), so it cannot possibly conflict with an actual desired default value.

</aside>

- Then, declare a way to compute this attribute **at the entity level**, through the `computeDefaults` property:

```tsx
const pokemonEntity = new EntityV2({
  schema: pokemonSchema,
  computeDefaults: {
    // 🙌 Correctly typed!
    levelPlusOne: ({ level }) => level + 1,
  },
});
```

In the tricky case of nested attributes, `computeDefaults` becomes an object with an `_attributes` or `_elements` property to emphasize that the computing is **local**:

```tsx

const pokemonSchema = schema({
  ...
  defaultLevel: number(),
	// 👇 Defaulted Map attribute
	levelHistory: map({
    currentLevel: number(),
		// 👇 Defaulted sub-attribute
		nextLevel: number().default(ComputedDefault)
	}).default(ComputedDefault),
})

const pokemonEntity = new EntityV2({
	// ...
	schema: pokemonSchema,
  computeDefaults: {
		levelHistory: {
      // Defaulted value of Map attribute
      _map: (item) => ({
        currentLevel: item.defaultLevel,
        nextLevel: item.defaultLevel
      }),
			_attributes: {
		   // Defaulted value of sub-attribute
				nextLevel: (levelHistory, item) => levelHistory.currentLevel + 1,
			},
	}
})
```

Note that there is (and has always been) an ambiguity as to when `default` values are actually used, that I hope to solve soon by splitting it into `getDefault`, `putDefault`, `updateDefault` and so on (`default` being the one to rule them all). For the moment, **`defaults` are only used in `putItem` commands.**

## Commands

Now that we know how to design entities, let’s take a look at how we can leverage them to craft commands 👍

<aside>
💡 *As stated in the intro, the beta only support the `PUT`, `GET`, and `DELETE` commands. If you need to run `UPDATE`, `QUERY` or `SCAN` commands, our advice is to run native SDK commands and format their output with the [`formatSavedItem` util](#formatsaveditem).*

</aside>

_[TO FINISH]_ v0.x did not favor tree-shaking.

```tsx
// v0.x Not tree-shakable
const response = await pokemonEntity.putItem(pokemonItem, options);
```

_[TO FINISH]_ favored tree-shaking.

```tsx
import { PutItemCommand } from 'dynamodb-toolbox';

// v1 Tree-shakable 🙌
const command = new PutItemCommand(
  pokemonEntity,
  pokemonItem,
  // 👇 Optional
  putItemOptions,
);

const params = command.params();
const response = await command.send();
```

Note that `pokemonItem`, can be provided later or edited, which can be useful if the command is a result of a computation. At execution, they will throw an error of no item has been provided:

```tsx
import { PutItemCommand } from 'dynamodb-toolbox';

const incompleteCommand = new PutItemCommand(pokemonEntity);

// (will return a new command and not mutate the original one)
const completeCommand = incompleteCommand.item(pokemonItem);

// (can be chained by design)
const response = incompleteCommand.item(pokemonItem).options(options).send();
```

You can also use the `.build` method of the entity to craft a command directly hydrated with your entity:

```tsx
// 🙌 We get a syntax closer to v0.x... but tree-shakable!
const response = await pokemonEntity
  .build(PutItemCommand)
  .item(pokemonItem)
  .options(options)
  .send();
```

<aside>
💡 *As much as I appreciate this syntax, it makes mocking hard in unit tests. I'm already working on a `mockEntity` helper, inspired by the awesome [`aws-sdk-client-mock`](https://github.com/m-radzikowski/aws-sdk-client-mock). This will probably make another article soon.*

</aside>

### PutItemCommand

_[TO FINISH]_ The `capacity`, `metrics` and `returnValues` options behave exactly the same as in the previous versions. The `condition` option benefit from improved typing, and clearer logical combinations:

```tsx
import { PutItemCommand } from "dynamodb-toolbox"

const { Attributes } = await pokemonEntity.build(PutItemCommand)
	.item(pokemonItem)
	.options({
    capacity: "TOTAL",
		metrics: "SIZE",
		// 👇 Will type the response `Attributes`
		returnValues: "ALL_OLD",
    condition: {
      or: [
				{ attr: "pokemonId", exists: false },
				// 🙌 "lte" is correcly typed
				{ attr: "level", lte: 99 },
				// 🙌 You can nest logical combinations
				{ and: [{ not: { ... } }, ...] }
			]
    },
  }).send()
```

### GetItemCommand

_[TO FINISH]_ Only key attributes are to be provided, and you can use `attributes` (with improved typing as well)

```tsx
import { GetItemCommand } from 'dynamodb-toolbox';

const { Item } = await pokemonEntity
  .build(GetItemCommand)
  .key({ pokemonClass: 'pikachu', pokemonId: '123' })
  .options({
    capacity: 'TOTAL',
    consistent: true,
    // 👇 Will type the response `Item`
    attributes: ['pokemonId', 'pokemonType', 'level'],
  })
  .send();
```

### DeleteItemCommand

_[TO FINISH]_ Similar to deleteItem (only key attributes have to be provided), and `getItem` (supports conditions)

```tsx
import { DeleteItemCommand } from "dynamodb-toolbox"

const { Attributes } = await pokemonEntity.build(DeleteItemCommand)
	.key({ pokemonClass: "pikachu", pokemonId: "123" })
	.options({
    capacity: "TOTAL",
		metrics: "SIZE",
		// 👇 Will type the response `Attributes`
		returnValues: "ALL_OLD",
    condition: {
      or: [
				{ attr: "level", lte: 99 },
				...
			]
    },
  }).send()
```

## Utility helpers and types

Beside of `SavedItem` and `FormattedItem`:

### formatSavedItem

_[TO FINISH]_ `formatSavedItem` transforms a raw item returned by DynamoDB to it’s formatted counterpart:

```tsx
import { formatSavedItem } from "dynamodb-toolbox"

// 🙌 Typed as FormattedItem<typeof pokemonEntity>
const formattedPokemon = formatSavedItem(
	pokemonEntity,
	savedPokemon,
	// As in GetItemCommand, attributes will filter the formatted item
	{ attributes: [...] }
)
```

Note that **it is a parsing operation**, i.e. it does not require the item to be typed as `SavedItem<typeof myEntity>`, but will throw an error if the saved item is invalid:

```tsx
const formattedPokemon = formatSavedItem(
	pokemonEntity,
	{ level: "not a number", ... },
)
// ❌ Will throw:
// => "Invalid attribute in saved item: level. Should be a number"
```

### Condition and parseCondition

_[TO FINISH]_

```tsx
import { Condition, parseCondition } from 'dynamodb-toolbox';

const condition: Condition<typeof pokemonEntity> = {
  attr: 'level',
  lte: 42,
};

const parsedCondition = parseCondition(pokemonEntity, condition);
// => {
//	ConditionExpression: "#1 <= :1",
//	ExpressionAttributeNames: { "#1": "level" },
//	ExpressionAttributeValues: { ":1": 42 }
// }
```

### Projection and parseProjection

_[TO FINISH]_

```tsx
import { AnyAttributePath, parseProjection } from 'dynamodb-toolbox';

const attributes: AnyAttributePath<typeof pokemonEntity>[] = [
  'pokemonType',
  'levelHistory.currentLevel',
];

const parsedCondition = parseProjection(pokemonEntity, attributes);
// => {
//	ProjectionExpression: '#1, #2.#3',
//	ExpressionAttributeNames: {
//	'#1': 'pokemonType',
//	'#2': 'levelHistory'
//	'#3': 'currentLevel'
//	}
// }
```

### KeyInput and PrimaryKey

_[TO FINISH]_

```tsx
import type { KeyInput, PrimaryKey } from 'dynamodb-toolbox';

type PokemonKeyInput = KeyInput<typeof pokemonEntity>;
// => { pokemonClass: string, pokemonId: string }

type MyTablePrimaryKey = PrimaryKey<typeof myTable>;
// => { PK: string, SK: string }
```

## Errors

Finally, let’s take a quick look at error management. When DynamoDB-Toolbox encounters an unexpected input, it will throw an instance of `DynamoDBToolboxError`, which itself extends the native `Error` class with a `code` property:

```tsx
await pokemonEntity
  .build(PutItemCommand)
  .item({ ..., level: 'not a number' })
  .send();
// ❌ [parsing.invalidAttributeInput] Attribute level should be a number
```

Some `DynamoDBToolboxErrors` also expose a `path` property (mostly in validations) and/or a `payload` property for additional context. If you need to handle them, TypeScript is your best friend, as the `code` property will correctly discriminate the `DynamoDBToolboxError` type:

```tsx
import { DynamoDBToolboxError } from "dynamodb-toolbox"

const handleError = (error: Error) => {
  if (!error instanceof DynamoDBToolboxError) throw error;

  switch (error.code) {
    case "parsing.invalidAttributeInput":
      const path = error.path
			 // => "level"
      const payload = error.payload
			// => { received: "not a number", expected: "number" }
			break
	    ...
    case "entity.invalidItemSchema":
      const path = error.path // ❌ error does not have path property
			const payload = error.payload // ❌ same goes with payload
	    ...
  }
}
```

## Conclusion

And that’s it for now! I hope you’re as excited as I am about this new release 🙌

_[TO FINISH]_ If have features that I missed in mind, or if you would like to see some of the ones I mentioned prioritised, please comment this article and/or [create an issue or open a discussion on the official repo](https://github.com/jeremydaly/dynamodb-toolbox) with the `v1` label 👍

See you soon!
