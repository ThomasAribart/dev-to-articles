---
published: true
title: 'New DynamoDB-Toolbox v1 beta: Features and breaking changes'
cover_image: https://raw.githubusercontent.com/ThomasAribart/dev-to-articles/master/blog-posts/dynamodb-toolbox-v1-beta/dynamodb-toolbox-v1-beta.png
description: 'New DynamoDB-Toolbox v1 beta: Features and breaking changes'
tags: Typescript, DynamoDB, AWS, Serverless
series:
canonical_url:
---

> ☝️ _NOTE: This article details how to migrate from the <code>beta.0</code> to the <code>beta.1</code> of the new DynamoDB-Toolbox major._
>
> 👉 _If you need documentation for the <code>beta.1</code> release, you may be looking for [this article](TODO)._
>
> 👉 _If you need documentation for the <code>beta.0</code> release, you may be looking for its [previous version](https://dev.to/slsbytheodo/the-dynamodb-toolbox-v1-beta-is-here-all-you-need-to-know-22op)._

A **new v1 beta of DynamoDB-Toolbox is out** 🙌

Simply named `beta.1`, the aim of this article is to succinctly describe its new features and breaking changes from the `beta.0`.

## New features

### UpdateItemCommand

The main update from this release is that the `UpdateItemCommand` is now available 🥳

It's clearly one of of the hardest feature I had to develop in my life (thank you, [DynamoDB reference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateItem.html) for being very unclear on soooooo many edge cases 😳). I'm glad it is now shipped!

See the [updated documentation article](TODO) for an exhaustive documentation, but here's a sneak peek:

```tsx
import { UpdateItemCommand, $add, $get } from 'dynamodb-toolbox';

const { Item } = await pokemonEntity
  .build(UpdateItemCommand)
  .item({
    pokemonId,
    ...
    name: newName,
    // Save current 'level' value to 'previousLevel'
    previousLevel: $get('level'),
    // Add 1 to the current 'level'
    level: $add(1),
  })
  .options({
    // Only if level is < 99
    condition: { attr: 'level', lt: 99 },
    // Will type the response `Attributes`
    returnValues: 'ALL_NEW',
    ...
  })
  .send();
```

### MockEntity

Also, a new `mockEntity` util is now available to help you write unit tests and make assertions:

```tsx
import { mockEntity } from 'dynamodb-toolbox';

const mockedPokemonEntity = mockEntity(pokemonEntity);

mockedPokemonEntity.on(GetItemCommand).resolve({
  // 🙌 Type-safe!
  Item: {
    pokemonId: 'pikachu1',
    name: 'Pikachu',
    level: 42,
    ...
  },
});

await pokemonEntity
  .build(GetItemCommand)
  .key({ pokemonId: 'pikachu1' })
  .options({ consistent: true })
  .send();
// => Will return mocked values!

mockedPokemonEntity.received(GetItemCommand).args(0);
// => [{ pokemonId: 'pikachu1' }, { consistent: true }]
```

### Providing getters for table names

Last small improvement: Table names can now be provided with getters. This can be useful in some contexts where you may want to use the `Table` class without actually running any command (e.g. tests or deployments):

```tsx
const myTable = new TableV2({
  ...
  // 👇 Only executed at command execution
  name: () => process.env.TABLE_NAME,
});
```

## Breaking changes

### `default` options

With the appearance of the `UpdateItemCommand`, the `default` option **had** to be reworked.

I disliked it for being ambiguous and impractical in some cases (like the internal `created` timestamp attribute). Well, it is now split into three options:

- `defaults.key` option _(`keyDefault` method)_: Fills undefined key attributes during key computing. Used in all commands for now, we'll see about `Queries` and `Scans`.
- `defaults.put` option _(`putDefault` method)_: Used for non-key attributes in `PutItemCommands`
- `defaults.update` option _(`updateDefault` method)_: Used for non-key attributes in `UpdateItemCommands`

```tsx
// 👇 Regular attribute
const lastUpdateAttribute = string({
  defaults: {
    key: undefined,
    put: () => new Date().toISOString(),
    update: () => new Date().toISOString(),
  },
});
// ...or
const lastUpdateAttribute = string()
  .putDefault(() => new Date().toISOString())
  .updateDefault(() => new Date().toISOString());

// 👇 Key attribute
const keyAttribute = string({
  key: true,
  defaults: {
    key: 'my-awesome-partition-key',
    // put & update defaults are not useful in `key` attributes
    put: undefined,
    update: undefined,
  },
});
// ...or
const keyAttribute = string().key().keyDefault('my-awesome-partition-key');
```

Note that the `default` method is still there. It acts similarly as `putDefault`, except if the attribute has been tagged as a `key` attribute, in which case it acts as `keyDefault`:

```tsx
const metadata = any().default({ any: 'value' });
// 👇 Similar to
const metadata = any().putDefault({ any: 'value' });
// 👇 ...or
const metadata = any({
  defaults: {
    key: undefined,
    put: { any: 'value' },
    update: undefined,
  },
});

const keyPart = any().key().default('my-awesome-partition-key');
// 👇 Similar to
const metadata = any().key().keyDefault('my-awesome-partition-key');
// 👇 ...or
const metadata = any({
  key: true,
  defaults: {
    key: 'my-awesome-partition-key',
    // put & update defaults are not useful in `key` attributes
    put: undefined,
    update: undefined,
  },
});
```

Also, the `const` shorthand still acts `enum(MY_CONST).default(MY_CONST)` and does NOT provide update defaults. Let me know if it's something you'd need.

### `computeDefaults`

The same split had to be applied to computed defaults: In the same spirit, the `computeDefaults` property has been renamed `putDefaults`, and a new `updateDefaults` property has appeared (with `computeKey` still being available to compute the primary key):

```tsx
import { schema, EntityV2, ComputedDefault } from 'dynamodb-toolbox';

const pokemonSchema = schema({
  ...
  level: number(),
  levelPlusOne: number().default(ComputedDefault),
  previousLevel: number().updateDefault(ComputedDefault),
});

const pokemonEntity = new EntityV2({
  ...
  schema: pokemonSchema,
  putDefaults: {
    // 🙌 Correctly typed!
    levelPlusOne: ({ level }) => level + 1,
  },
  updateDefaults: {
    // 🙌 Correctly typed!
    previousLevel: ({ level }) =>
      // Update 'previousLevel' only if 'level' is updated
      level !== undefined ? $get('level') : undefined,
  },
});
```

## Conclusion

That's it 🙌 I hope you like this new version... and YES, `Queries` and `Scans` are next!

See you in a few months for the official release! (before the end of the year I hope 🤞)
