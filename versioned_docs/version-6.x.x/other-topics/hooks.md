---
title: Hooks
---

Hooks (also known as lifecycle events), are functions which are called before and after calls in sequelize are executed. For example, if you want to always set a value on a model before saving it, you can add a `beforeUpdate` hook.

**Note:** _You can't use hooks with instances. Hooks are used with models._

## Available hooks

Sequelize provides a lot of hooks. The full list can be found in directly in the [source code - src/hooks.js](https://github.com/sequelize/sequelize/blob/v6/src/hooks.js#L7).

## Hooks firing order

The diagram below shows the firing order for the most common hooks.

_**Note:** this list is not exhaustive._

```text
(1)
  beforeBulkCreate(instances, options)
  beforeBulkDestroy(options)
  beforeBulkUpdate(options)
(2)
  beforeValidate(instance, options)

[... validation happens ...]

(3)
  afterValidate(instance, options)
  validationFailed(instance, options, error)
(4)
  beforeCreate(instance, options)
  beforeDestroy(instance, options)
  beforeUpdate(instance, options)
  beforeSave(instance, options)
  beforeUpsert(values, options)

[... creation/update/destruction happens ...]

(5)
  afterCreate(instance, options)
  afterDestroy(instance, options)
  afterUpdate(instance, options)
  afterSave(instance, options)
  afterUpsert(created, options)
(6)
  afterBulkCreate(instances, options)
  afterBulkDestroy(options)
  afterBulkUpdate(options)
```

## Declaring Hooks

Arguments to hooks are passed by reference. This means, that you can change the values, and this will be reflected in the insert / update statement. A hook may contain async actions - in this case the hook function should return a promise.

There are currently three ways to programmatically add hooks:

```js
// Method 1 via the .init() method
class User extends Model {}
User.init({
  username: DataTypes.STRING,
  mood: {
    type: DataTypes.ENUM,
    values: ['happy', 'sad', 'neutral']
  }
}, {
  hooks: {
    beforeValidate: (user, options) => {
      user.mood = 'happy';
    },
    afterValidate: (user, options) => {
      user.username = 'Toni';
    }
  },
  sequelize
});

// Method 2 via the .addHook() method
User.addHook('beforeValidate', (user, options) => {
  user.mood = 'happy';
});

User.addHook('afterValidate', 'someCustomName', (user, options) => {
  return Promise.reject(new Error("I'm afraid I can't let you do that!"));
});

// Method 3 via the direct method
User.beforeCreate(async (user, options) => {
  const hashedPassword = await hashPassword(user.password);
  user.password = hashedPassword;
});

User.afterValidate('myHookAfter', (user, options) => {
  user.username = 'Toni';
});
```

## Removing hooks

Only a hook with name param can be removed.

```js
class Book extends Model {}
Book.init({
  title: DataTypes.STRING
}, { sequelize });

Book.addHook('afterCreate', 'notifyUsers', (book, options) => {
  // ...
});

Book.removeHook('afterCreate', 'notifyUsers');
```

You can have many hooks with same name. Calling `.removeHook()` will remove all of them.

## Global / universal hooks

Global hooks are hooks that are run for all models. They are especially useful for plugins and can define behaviours that you want for all your models, for example to allow customization on timestamps using `sequelize.define` on your models:

```js
const User = sequelize.define('User', {}, {
    tableName: 'users',
    hooks : {
        beforeCreate : (record, options) => {
            record.dataValues.createdAt = new Date().toISOString().replace(/T/, ' ').replace(/\..+/g, '');
            record.dataValues.updatedAt = new Date().toISOString().replace(/T/, ' ').replace(/\..+/g, '');
        },
        beforeUpdate : (record, options) => {
            record.dataValues.updatedAt = new Date().toISOString().replace(/T/, ' ').replace(/\..+/g, '');
        }
    }
});
```

They can be defined in many ways, which have slightly different semantics:

### Default Hooks (on Sequelize constructor options)

```js
const sequelize = new Sequelize(..., {
  define: {
    hooks: {
      beforeCreate() {
        // Do stuff
      }
    }
  }
});
```

This adds a default hook to all models, which is run if the model does not define its own `beforeCreate` hook:

```js
const User = sequelize.define('User', {});
const Project = sequelize.define('Project', {}, {
  hooks: {
    beforeCreate() {
      // Do other stuff
    }
  }
});

await User.create({});    // Runs the global hook
await Project.create({}); // Runs its own hook (because the global hook is overwritten)
```

### Permanent Hooks (with `sequelize.addHook`)

```js
sequelize.addHook('beforeCreate', () => {
  // Do stuff
});
```

This hook is always run, whether or not the model specifies its own `beforeCreate` hook. Local hooks are always run before global hooks:

```js
const User = sequelize.define('User', {});
const Project = sequelize.define('Project', {}, {
  hooks: {
    beforeCreate() {
      // Do other stuff
    }
  }
});

await User.create({});    // Runs the global hook
await Project.create({}); // Runs its own hook, followed by the global hook
```

Permanent hooks may also be defined in the options passed to the Sequelize constructor:

```js
new Sequelize(..., {
  hooks: {
    beforeCreate() {
      // do stuff
    }
  }
});
```

Note that the above is not the same as the *Default Hooks* mentioned above. That one uses the `define` option of the constructor. This one does not.

### Connection Hooks

Sequelize provides four hooks that are executed immediately before and after a database connection is obtained or released:

* `sequelize.beforeConnect(callback)`
  * The callback has the form `async (config) => /* ... */`
* `sequelize.afterConnect(callback)`
  * The callback has the form `async (connection, config) => /* ... */`
* `sequelize.beforeDisconnect(callback)`
  * The callback has the form `async (connection) => /* ... */`
* `sequelize.afterDisconnect(callback)`
  * The callback has the form `async (connection) => /* ... */`

These hooks can be useful if you need to asynchronously obtain database credentials, or need to directly access the low-level database connection after it has been created.

For example, we can asynchronously obtain a database password from a rotating token store, and mutate Sequelize's configuration object with the new credentials:

```js
sequelize.beforeConnect(async (config) => {
  config.password = await getAuthToken();
});
```

These hooks may *only* be declared as a permanent global hook, as the connection pool is shared by all models.

## Instance hooks

The following hooks will emit whenever you're editing a single object:

* `beforeValidate`
* `afterValidate` / `validationFailed`
* `beforeCreate` / `beforeUpdate` / `beforeSave` / `beforeDestroy`
* `afterCreate` / `afterUpdate` / `afterSave` / `afterDestroy`

```js
User.beforeCreate(user => {
  if (user.accessLevel > 10 && user.username !== "Boss") {
    throw new Error("You can't grant this user an access level above 10!");
  }
});
```

The following example will throw an error:

```js
try {
  await User.create({ username: 'Not a Boss', accessLevel: 20 });
} catch (error) {
  console.log(error); // You can't grant this user an access level above 10!
};
```

The following example will be successful:

```js
const user = await User.create({ username: 'Boss', accessLevel: 20 });
console.log(user); // user object with username 'Boss' and accessLevel of 20
```

### Model hooks

Sometimes you'll be editing more than one record at a time by using methods like `bulkCreate`, `update` and `destroy`. The following hooks will emit whenever you're using one of those methods:

* `YourModel.beforeBulkCreate(callback)`
  * The callback has the form `(instances, options) => /* ... */`
* `YourModel.beforeBulkUpdate(callback)`
  * The callback has the form `(options) => /* ... */`
* `YourModel.beforeBulkDestroy(callback)`
  * The callback has the form `(options) => /* ... */`
* `YourModel.afterBulkCreate(callback)`
  * The callback has the form `(instances, options) => /* ... */`
* `YourModel.afterBulkUpdate(callback)`
  * The callback has the form `(options) => /* ... */`
* `YourModel.afterBulkDestroy(callback)`
  * The callback has the form `(options) => /* ... */`

Note: methods like `bulkCreate` do not emit individual hooks by default - only the bulk hooks. However, if you want individual hooks to be emitted as well, you can pass the `{ individualHooks: true }` option to the query call. However, this can drastically impact performance, depending on the number of records involved (since, among other things, all instances will be loaded into memory). Examples:

```js
await Model.destroy({
  where: { accessLevel: 0 },
  individualHooks: true
});
// This will select all records that are about to be deleted and emit `beforeDestroy` and `afterDestroy` on each instance.

await Model.update({ username: 'Tony' }, {
  where: { accessLevel: 0 },
  individualHooks: true
});
// This will select all records that are about to be updated and emit `beforeUpdate` and `afterUpdate` on each instance.
```

If you use `Model.bulkCreate(...)` with the `updateOnDuplicate` option, changes made in the hook to fields that aren't given in the `updateOnDuplicate` array will not be persisted to the database. However it is possible to change the `updateOnDuplicate` option inside the hook if this is what you want.

```js
User.beforeBulkCreate((users, options) => {
  for (const user of users) {
    if (user.isMember) {
      user.memberSince = new Date();
    }
  }

  // Add `memberSince` to updateOnDuplicate otherwise it won't be persisted
  if (options.updateOnDuplicate && !options.updateOnDuplicate.includes('memberSince')) {
    options.updateOnDuplicate.push('memberSince');
  }
});

// Bulk updating existing users with updateOnDuplicate option
await Users.bulkCreate([
  { id: 1, isMember: true },
  { id: 2, isMember: false }
], {
  updateOnDuplicate: ['isMember']
});
```

## Exceptions

Only __Model methods__ trigger hooks. This means there are a number of cases where Sequelize will interact with the database without triggering hooks.
These include but are not limited to:

- Instances being deleted by the database because of an `ON DELETE CASCADE` constraint, [except if the `hooks` option is true](#hooks-for-cascade-deletes).
- Instances being updated by the database because of a `SET NULL` or `SET DEFAULT` constraint.
- [Raw queries](../core-concepts/raw-queries.md).
- All QueryInterface methods.

If you need to react to these events, consider using your database's native triggers and notification system instead.

## Hooks for cascade deletes

As indicated in [Exceptions](#exceptions), Sequelize will not trigger hooks when instances are deleted by the database because of an `ON DELETE CASCADE` constraint.

However, if you set the `hooks` option to `true` when defining your association, Sequelize will trigger the `beforeDestroy` and `afterDestroy` hooks for the deleted instances.

:::caution

Using this option is discouraged for the following reasons:

- This option requires many extra queries. The `destroy` method normally executes a single query.
  If this option is enabled, an extra `SELECT` query, as well as an extra `DELETE` query for each row returned by the select will be executed.
- If you do not run this query in a transaction, and an error occurs, you may end up with some rows deleted and some not deleted.
- This option only works when the *instance* version of `destroy` is used. The static version will not trigger the hooks, even with `individualHooks`.
- This option will not work in `paranoid` mode.
- This option will not work if you only define the association on the model that owns the foreign key. You need to define the reverse association as well.

This option is considered legacy. We highly recommend using your database's triggers and notification system if you need to be notified of database changes.

:::

Here is an example of how to use this option:

```ts
import { Model } from 'sequelize';

const sequelize = new Sequelize({ /* options */ });

class User extends Model {}

User.init({}, { sequelize });

class Post extends Model {}

Post.init({}, { sequelize });
Post.beforeDestroy(() => {
  console.log('Post has been destroyed');
});

// This "hooks" option will cause the "beforeDestroy" and "afterDestroy"
// highlight-next-line
User.hasMany(Post, { onDelete: 'cascade', hooks: true });

await sequelize.sync({ force: true });

const user = await User.create();
const post = await Post.create({ userId: user.id });

// this will log "Post has been destroyed"
await user.destroy();
```

## Associations

For the most part hooks will work the same for instances when being associated.

### One-to-One and One-to-Many associations

* When using `add`/`set` mixin methods the `beforeUpdate` and `afterUpdate` hooks will run.

### Many-to-Many associations

* When using `add` mixin methods for `belongsToMany` relationships (that will add one or more records to the junction table) the `beforeBulkCreate` and `afterBulkCreate` hooks in the junction model will run.
  * If `{ individualHooks: true }` was passed to the call, then each individual hook will also run.

* When using `remove` mixin methods for `belongsToMany` relationships (that will remove one or more records to the junction table) the `beforeBulkDestroy` and `afterBulkDestroy` hooks in the junction model will run.
  * If `{ individualHooks: true }` was passed to the call, then each individual hook will also run.

If your association is Many-to-Many, you may be interested in firing hooks on the through model when using the `remove` call. Internally, sequelize is using `Model.destroy` resulting in calling the `bulkDestroy` instead of the `before/afterDestroy` hooks on each through instance.

## Hooks and Transactions

Many model operations in Sequelize allow you to specify a transaction in the options parameter of the method. If a transaction *is* specified in the original call, it will be present in the options parameter passed to the hook function. For example, consider the following snippet:

```js
User.addHook('afterCreate', async (user, options) => {
  // We can use `options.transaction` to perform some other call
  // using the same transaction of the call that triggered this hook
  await User.update({ mood: 'sad' }, {
    where: {
      id: user.id
    },
    transaction: options.transaction
  });
});

await sequelize.transaction(async t => {
  await User.create({
    username: 'someguy',
    mood: 'happy'
  }, {
    transaction: t
  });
});
```

If we had not included the transaction option in our call to `User.update` in the preceding code, no change would have occurred, since our newly created user does not exist in the database until the pending transaction has been committed.

### Internal Transactions

It is very important to recognize that sequelize may make use of transactions internally for certain operations such as `Model.findOrCreate`. If your hook functions execute read or write operations that rely on the object's presence in the database, or modify the object's stored values like the example in the preceding section, you should always specify `{ transaction: options.transaction }`:

* If a transaction was used, then `{ transaction: options.transaction }` will ensure it is used again;
* Otherwise, `{ transaction: options.transaction }` will be equivalent to `{ transaction: undefined }`, which won't use a transaction (which is ok).

This way your hooks will always behave correctly.
