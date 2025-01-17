---
title: Cross-Stack References
description: "Managing cross-stack references in your Serverless Stack (SST) app."
---

import TabItem from "@theme/TabItem";
import MultiLanguageCode from "@site/src/components/MultiLanguageCode";

One of the more powerful features of CDK is, automatic cross-stack references. When you pass a construct from one [`Stack`](../constructs/Stack.md) to another stack and reference it there; CDK will create a stack export with an auto-generated export name in the stack with the construct. And then import that value in the stack that's referencing it.

## Adding a reference

So imagine you have a DynamoDB [`Table`](../constructs/Table.md) in one stack, and you need to add the table name as an environment variable (for the Lambda functions) in another stack.

To do this, start by exposing the table as a class property.

<MultiLanguageCode>
<TabItem value="js">

```js {7-12} title="stacks/StackA.js"
import { Stack, Table, TableFieldType } from "@serverless-stack/resources";

export class StackA extends Stack {
  constructor(scope, id) {
    super(scope, id);

    this.table = new Table(this, "MyTable", {
      fields: {
        pk: TableFieldType.STRING,
      },
      primaryIndex: { partitionKey: "pk" },
    });
  }
}
```

</TabItem>
<TabItem value="ts">

```js {9-14} title="stacks/StackA.ts"
import { App, Stack, Table, TableFieldType } from "@serverless-stack/resources";

export class StackA extends Stack {
  public readonly table: Table;

  constructor(scope: App, id: string) {
    super(scope, id);

    this.table = new Table(this, "MyTable", {
      fields: {
        pk: TableFieldType.STRING,
      },
      primaryIndex: { partitionKey: "pk" },
    });
  }
}
```

</TabItem>
</MultiLanguageCode>

Then pass the table to `StackB`.

<MultiLanguageCode>
<TabItem value="js">

```js {3} title="stacks/index.js"
const stackA = new StackA(app, "StackA");

new StackB(app, "StackB", stackA.table);
```

</TabItem>
<TabItem value="ts">

```js {3} title="stacks/index.ts"
const stackA = new StackA(app, "StackA");

new StackB(app, "StackB", stackA.table);
```

</TabItem>
</MultiLanguageCode>

Finally, reference the table's name in `StackB`.

<MultiLanguageCode>
<TabItem value="js">

```js {10} title="stacks/StackB.js"
import { Api, Stack } from "@serverless-stack/resources";

export class StackB extends Stack {
  constructor(scope, id, table) {
    super(scope, id);

    new Api(this, "Api", {
      defaultFunctionProps: {
        environment: {
          TABLE_NAME: table.tableName,
        },
      },
      routes: {
        "GET /": "src/lambda.main",
      },
    });
  }
}
```

</TabItem>
<TabItem value="ts">

```js {10} title="stacks/StackB.ts"
import { Api, App, Stack, Table } from "@serverless-stack/resources";

export class StackB extends Stack {
  constructor(scope: App, id: string, table: Table) {
    super(scope, id);

    new Api(this, "Api", {
      defaultFunctionProps: {
        environment: {
          TABLE_NAME: table.tableName,
        },
      },
      routes: {
        "GET /": "src/lambda.main",
      },
    });
  }
}
```

</TabItem>
</MultiLanguageCode>

Behind the scenes, the table name is exported as an output of `StackA`. If you head over to your AWS CloudFormation console and look at `StackA`'s outputs, you should see an output with:

- The key prefixed with `ExportsOutput`. Something like `ExportsOutputRefMyTableCD79AAA0A1504A18`.
- The value of the table name, ie. `dev-demo-StackA-MyTableCD79AAA0-1CUTVKQQ47K31`.
- And the export name will include the stack name and the key, ie. `dev-demo-StackA:ExportsOutputRefMyTableCD79AAA0A1504A18`.

And if you look at `StackB`'s template, you should see the output being imported using the `Fn::ImportValue` intrinsic notation. For example, `Fn::ImportValue: dev-demo-StackA:ExportsOutputRefMyTableCD79AAA0A1504A18`.

## Order of deployment

By doing this, it will add a dependency between the stacks. And the stack exporting the value will be deployed before the stack importing it. In the example above, `StackA` will be deployed first, and then `StackB` will be deployed later.

## Removing a reference

Now suppose in the example above, `StackB` no longer needs the table name as a Lambda environment variable. So we remove the `environment` option and change the `Api` to:

```js
new Api(this, "Api", {
  routes: {
    "GET /": "src/lambda.main",
  },
});
```

When you try to deploy your app, you'll likely get an `Export XXXX cannot be deleted` error. It'll look similar to this:

```
dev-demo-StackA Export dev-demo-StackA:ExportsOutputRefMyTableCD79AAA0A1504A18 cannot be deleted as it is in use by dev-demo-StackB
```

This will happen because:

- With the cross-stack references removed, `StackB` no long depends on `StackA`, and both stacks will get deployed concurrently.
- While they are both being deployed, AWS will find that `StackA`'s output cannot be removed because `StackB` is still importing it.

To fix this, we need to first remove `StackB`'s dependency on `StackA`, deploy it, then remove the export. It'll be a 2-step process:

1. After we remove the reference in `StackB`, we'll tell CDK that we still want the output exported in `StackA`. We can do this by explicitly calling `this.exportValue`.

   ```js {12} title="stacks/StackA.js"
   export class StackA extends Stack {
     constructor(scope, id) {
       super(scope, id);
   
       this.table = new Table(this, "MyTable", {
         fields: {
           pk: TableFieldType.STRING,
         },
         primaryIndex: { partitionKey: "pk" },
       });

       this.exportValue(this.table.tableName);
     }
   }
   ```
 
   **Deploy.**

   This changes the reference in `StackB` but leaves `StackA` as-is.

2. After `StackB` finishes deploying, `StackA`'s export is no longer being imported. So you can remove the `this.exportValue` line.

   **And deploy again.**

### Automatic export injection

You shouldn't have to perform the two-step process above, starting from v0.60.8 SST will handle this automatically.

Prior to deploying the CloudFormation stack, SST will look for exports that are about to be removed but are still being imported by other stacks. If such exports are found, SST will automatically inject the export into the CloudFormation template. In the example above, SST will create the export `dev-demo-StackA:ExportsOutputRefMyTableCD79AAA0A1504A18` with the same value as that is currently being used. This has the same effect as the step 1 above.

And when you deploy again, since the export `dev-demo-StackA:ExportsOutputRefMyTableCD79AAA0A1504A18` is no longer being used in other stacks, SST will not inject it to the template.
