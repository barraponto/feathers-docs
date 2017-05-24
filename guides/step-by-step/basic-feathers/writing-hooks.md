# Writing Your Own Hooks


## Hook function template

Hook functions should be written like this:
```javascript
// Outer function initializes the hook function
function myHook(options) {
  // The hook function itself is returned.
  return context => {
    // You can use the options param to condition behavior within the hook.
  };
}
```

Feathers calls the inner function with the [context object](../../../api/hooks.md#hook-objects).

`context.result` is an object or array for all method calls other than `find`.
It is [an object](../../../api/databases/common.html#pagination) if the `find` is paginated.
Otherwise it is an array.

The hook function may either return synchronously or it may return a Promise.
The return value (sync) or resolved value from the Promise (async) may be either
a new context object, or `undefined`.

> **ProTip** The context object is not changed if `undefined` is returned.



> **ProTip** Mutating the `context` param inside a hook function
does not change the context object passed to the next hook.

Let's review the source of some of the [common hooks](../../../api/hooks-common.md)
to learn how to write our own.


## debug [source](https://github.com/feathersjs/feathers-hooks-common/blob/master/src/services/debug.js)

`debug` logs the context to the console.

```javascript
export default function (msg) {
  return context => {
    console.log(`* ${msg || ''}\ntype:${context.type}, method: ${context.method}`);
    if (context.data) { console.log('data:', context.data); }
    if (context.params && context.params.query) { console.log('query:', context.params.query); }
    if (context.result) { console.log('result:', context.result); }
    if (context.error) { console.log('error', context.error); }
  };
}
```

This hook is straightforward, simply displaying some of the `context` object properties.
The context object does not change as the inner hook function returns `undefined` by default.

`debug` is great for debugging other hooks.
Once you place this hook before and after the hook under test,
you'll see the context object the test hook received, and what it returned.

This example
- Shows several context properties.
- Leaves the context object unchanged with a sync `return`.


## disableMultiItemChange [source](https://github.com/feathersjs/feathers-hooks-common/blob/master/src/services/disable-multi-item-change.js)

`disableMultiItemChange` disables update, patch and remove methods from using `null` as an id,
e.g. `remove(null)`.
A `null` id affects all the items in the DB, so accidentally using it may have undesirable results.

```javascript
import errors from 'feathers-errors';
import checkContext from './check-context';

export default function () {
  return function (context) {
    checkContext(context, 'before', ['update', 'patch', 'remove'], 'disableMultiItemChange');

    if (context.id === null) {
      throw new errors.BadRequest(
        `Multi-record changes not allowed for ${context.path} ${context.method}. (disableMultiItemChange)`
      );
    }
  };
}
```

Some hooks may only be used `before` or `after`; some may be used only with certain methods.
The [`checkContext` utility](../../../api/hooks-common.md#util-checkcontext)
checks the hook function is being used properly.

This hook throws an error that will be properly returned to the client.
```javascript
service.patch(null, data, { query: { dept: 'acct' } })
  .then(data => ...)
  .catch(err => {
    console.log(err.message); // Multi-record changes not allowed for ...
  });
```

This example
- Introduces `checkContext`.
- Shows how to throw an error in hooks.


## pluckQuery [source](https://github.com/feathersjs/feathers-hooks-common/blob/master/src/services/pluck-query.js)

`pluckQuery` discards all fields from the query params except for the specified ones.
This helps sanitize the query.

```javascript
import _pluck from '../common/_pluck';
import checkContext from './check-context';

export default function (...fieldNames) {
  return context => {
    checkContext(context, 'before', null, 'pluckQuery');

    const query = (context.params || {}).query || {};
    context.params.query = _pluck(query, fieldNames);

    return context;
  };
}
```

The `_pluck` utility, given an object and an array of property name,
returns an object consisting of just those peoperties.
The property names may be in dot notation, e.g. `destination.address.city`.

The context object is modified and returned,
thus modifying what context is passed to the next hook.

This example
- Introduces `_pluck`.
- Modifies and synchronously returns the context object.


## pluck [source](https://github.com/feathersjs/feathers-hooks-common/blob/master/src/services/pluck.js)

`pluck` discards all fields except for the specified ones,
either from the data submitted or from the result.
If the data is an array or a paginated find result the hook will remove the field(s) for every item.

```javascript
import _pluck from '../common/_pluck';
import checkContextIf from './check-context-if';
import getItems from './get-items';
import replaceItems from './replace-items';

export default function (...fieldNames) {
  return context => {
    checkContextIf(context, 'before', ['create', 'update', 'patch'], 'pluck');

    if (context.params.provider) {
      replaceItems(context, _pluck(getItems(context), fieldNames));
    }

    return context;
  };
}
```

The [`getItems` utility](../../../api/hooks-common.md#util-getitems-replaceitems)
returns the items in either `hook.data` or `hook.result`
depending on whether the hook is being used as a before or after hook.
`hook.result.data` or `hook.result` is returned for a `find` method.

The returned items are always an array to simplify further processing.

The [`replaceItems` utility](../../../api/hooks-common.md#util-getitems-replaceitems)
is the reverse of `getItems`, returning the items where they came from.

This example
- Introduces the convenient `getItems` and `replaceItems` utilities.


## Simple async hook

Let's write a hook which will call an async function which returns a Promise.
That function will determine if the values in `hook.data` are valid.

```javascript
import errors from 'feathers-errors';

export default function (validator) {
  return context => {
    return validator(context.data)
      .then(() => context)
      .catch(errs => {
        throw new errors.BadRequest('Validation error', { errors: errs });
      });
  };
}
```

The hook either returns a Promise which resolves to the existing context object,
or it throws with an error object.

The next hook will not be run until this Promise resolves and this hook logically ends.

> **ProTip** Perhaps the most common error made when writing async hooks
is to *not return* the Promise.
The hook will not work as expected with `validator(context.data)`.
It must be `return validator(context.data)`.

This example
- Shows how to code async hooks.


## stashBefore [source](https://github.com/feathersjs/feathers-hooks-common/blob/master/src/services/stash-before.js)

`stashBefore` saves the current value of record before mutating it.

```javascript
import errors from 'feathers-errors';
import checkContext from './check-context';

export default function (prop) {
  const beforeField = prop || 'before';

  return context => {
    checkContext(context, 'before', ['get', 'update', 'patch', 'remove'], 'stashBefore');

    if (context.id === null || context.id === undefined) {
      throw new errors.BadRequest('Id is required. (stashBefore)');
    }

    if (context.params.query && context.params.query.$disableStashBefore) {
      delete context.params.query.$disableStashBefore;
      return context;
    }

    const params = (context.method === 'get') ? context.params : {
      provider: context.params.provider,
      authenticated: context.params.authenticated,
      user: context.params.user
    };

    params.query = params.query || {};
    params.query.$disableStashBefore = true;

    return context.service.get(context.id, params)
      .then(data => {
        delete params.query.$disableStashBefore;

        context.params[beforeField] = data;
        return context;
      })
      .catch(() => context);
  };
}
```

This hook is more complex than the previous ones.
Let's look at some of its key parts.

This is what the hook returns.

```javascript
return context.service.get(context.id, params)
  .then(data => {
    delete params.query.$disableStashBefore;

    context.params[beforeField] = data;
    return context;
  })
  .catch(() => context);
```

- `context.service` is always the current service.
- `context.service.get()` is an async call, and it returns a Promise.
- The hook returns that Promise, so its an async hook.
The next hook will only run once this Promise resolves.
- The data obtained by the `get` is placed into `context.params`.
- We can see the Promise will always resolve to `context`.

In summary, the hook will `get` the record being mutated by the call,
will place that record in `context.params`,
and will return the possibly modified `context`.
The method call will continue as if nothing has happened.

You always need to consider what value to assign `params` when calling a service within a hook.
If you don't assign a value, the `get` will run as being called on the server
(it is being called by the server after all)
even if the method call originated on a client.

This may not be OK.
The user password may be returned when a user record is read by the server,
but you would not want a client to have access to it.

There may be other issues if, for example, you are using Sequelize and the method call
includes parameters that are passed through to Sequelize.
What may be appropriate for an `update` may not be acceptable for a `get`.

`stashBefore` handles this with

```javascript
const params = context.method === 'get' ? context.params : {
  provider: context.params.provider,
  authenticated: context.params.authenticated,
  user: context.params.user
};
```

- On a `get` call we will use the same `params` for our inner `get`.
- On other calls, we use something "safe".
    - We copy over `provider` so our inner `get` acts like it has the same transport.
    - We copy standard authentication values for auth hooks.

Will this satisfy every use case?
No, but it will satisfy most.
You can always fork the hook and customize it.

> **ProTip** Setting `provider` is common and you should always consider doing so.

There is one more thing to consider.
The `stashBefore` hook will run again when we call the inner `get`.
This will cause a recursion of inner `get` calls unless we do something.

```javascript
if (context.params.query && context.params.query.$disableStashBefore) {
  delete context.params.query.$disableStashBefore;
  return context;
}
//
params.query = params.query || {};
params.query.$disableStashBefore = true;
//
delete params.query.$disableStashBefore;
```

We set a flag to show we are calling the inner `get`.
`stashBefore` will see the flag when it runs for that inner `get` and exit,
preventing recursion.

> **ProTip** Its not uncommon to indicate what state operations are in
by setting flags in `params`.

This example
- Shows how to call the current service.
- Discusses how to handle `params` for service calls.
- Shows how to prevent recursion.


## Using other services

You can get a handle to another service, while in a hook, with `context.app.service(path)`.

It may be easier, frankly, to use a service other than the current one,
as you are less likely to have to worry about recursion.

 
### Is anything wrong, unclear, missing?
[Leave a comment.](https://github.com/feathersjs/feathers-docs/issues/new?title=Comment:Step-Basic-Writing-Hooks&body=Comment:Step-Basic-Writing-Hooks)