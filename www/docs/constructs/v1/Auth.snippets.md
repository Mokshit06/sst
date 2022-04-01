### Allowing users to sign in using User Pool

```js
import { Auth } from "@serverless-stack/resources";

new Auth(this, "Auth", {
  cognito: true,
});
```

### Allowing users to sign in with their email or phone number

```js
new Auth(this, "Auth", {
  cognito: {
    cdk: {
      userPool: {
        signInAliases: { email: true, phone: true },
      },
    },
  },
});
```

### Configuring User Pool triggers

The Cognito User Pool can invoke a Lambda function for specific [triggers](#triggers).

#### Adding triggers

```js
new Auth(this, "Auth", {
  cognito: {
    triggers: {
      preAuthentication: "src/preAuthentication.main",
      postAuthentication: "src/postAuthentication.main",
    },
  },
});
```

#### Specifying function props for all the triggers

```js
new Auth(this, "Auth", {
  cognito: {
    defaults: {
      function: {
        timeout: 20,
        environment: { tableName: table.tableName },
        permissions: [table],
      },
    },
    triggers: {
      preAuthentication: "src/preAuthentication.main",
      postAuthentication: "src/postAuthentication.main",
    },
  },
});
```

#### Using the full config for a trigger

Configure each Lambda function separately.

```js
new Auth(this, "Auth", {
  cognito: {
    triggers: {
      preAuthentication: {
        handler: "src/preAuthentication.main",
        timeout: 10,
        environment: { bucketName: bucket.bucketName },
        permissions: [bucket],
      },
      postAuthentication: "src/postAuthentication.main",
    },
  },
});
```

Note that, you can set the `defaults.function` while using the `FunctionProps` per trigger. The `function` will just override the `defaults.function`. Except for the `environment`, the `layers`, and the `permissions` properties, it will be merged.

```js
new Auth(this, "Auth", {
  cognito: {
    defaults: {
      function: {
        timeout: 20,
        environment: { tableName: table.tableName },
        permissions: [table],
      },
    },
    triggers: {
      preAuthentication: {
        handler: "src/preAuthentication.main",
        timeout: 10,
        environment: { bucketName: bucket.bucketName },
        permissions: [bucket],
      },
      postAuthentication: "src/postAuthentication.main",
    },
  },
});
```

So in the above example, the `preAuthentication` function doesn't use the `timeout` that is set in the `defaults.function`. It'll instead use the one that is defined in the function definition (`10 seconds`). And the function will have both the `tableName` and the `bucketName` environment variables set; as well as permissions to both the `table` and the `bucket`.

#### Attaching permissions for all triggers

Allow all the triggers to access S3.

```js {10}
const auth = new Auth(this, "Auth", {
  cognito: {
    triggers: {
      preAuthentication: "src/preAuthentication.main",
      postAuthentication: "src/postAuthentication.main",
    },
  },
});

auth.attachPermissionsForTriggers(["s3"]);
```

#### Attaching permissions for a specific trigger

Allow one of the triggers to access S3.

```js {10}
const auth = new Auth(this, "Auth", {
  cognito: {
    triggers: {
      preAuthentication: "src/preAuthentication.main",
      postAuthentication: "src/postAuthentication.main",
    },
  },
});

auth.attachPermissionsForTriggers("preAuthentication", ["s3"]);
```

Here we are referring to the trigger using the trigger key, `preAuthentication`. 

### Allowing Twitter auth and a User Pool

```js
new Auth(this, "Auth", {
  cognito: true,
  twitter: {
    consumerKey: "gyMbPOiwefr6x63SjIW8NN2d9",
    consumerSecret: "qxld1zic5c2eyahqK3gjGLGQaOTogGfAgGh17MYOIcOUR9l2Nz",
  },
});
```

### Adding all the supported social logins

```js
new Auth(this, "Auth", {
  facebook: { appId: "419718329085014" },
  apple: { servicesId: "com.myapp.client" },
  amazon: { appId: "amzn1.application.24ebe4ee4aef41e5acff038aee2ee65f" },
  google: {
    clientId:
      "38017095028-abcdjaaaidbgt3kfhuoh3n5ts08vodt3.apps.googleusercontent.com",
  },
});
```

### Allowing users to login using Auth0

```js
new Auth(this, "Auth", {
  auth0: {
    domain: "https://myorg.us.auth0.com",
    clientId: "UsGRQJJz5sDfPQDs6bhQ9Oc3hNISuVif",
  },
});
```

### Attaching permissions for authenticated users

```js {7}
const auth = new Auth(this, "Auth", {
  cognito: {
    userPool: { signInAliases: { email: true } },
  },
});

auth.attachPermissionsForAuthUsers([api, "s3"]);
```

### Attaching permissions for unauthenticated users

```js {7}
const auth = new Auth(this, "Auth", {
  cognito: {
    userPool: { signInAliases: { email: true } },
  },
});

auth.attachPermissionsForUnauthUsers([api, "s3"]);
```

### Sharing Auth across stacks

You can create the Auth construct in one stack, and attach permissions in other stacks. To do this, expose the Auth as a class property.

<MultiLanguageCode>
<TabItem value="js">

```js {7-9} title="stacks/AuthStack.js"
import { Auth, Stack } from "@serverless-stack/resources";

export class AuthStack extends Stack {
  constructor(scope, id) {
    super(scope, id);

    this.auth = new Auth(this, "Auth", {
      cognito: true,
    });
  }
}
```

</TabItem>
<TabItem value="ts">

```js {4,9-11} title="stacks/AuthStack.ts"
import { App, Auth, Stack } from "@serverless-stack/resources";

export class AuthStack extends Stack {
  public readonly auth: Auth;

  constructor(scope: App, id: string) {
    super(scope, id);

    this.auth = new Auth(this, "Auth", {
      cognito: true,
    });
  }
}
```

</TabItem>
</MultiLanguageCode>

Then pass the Auth to a different stack.

<MultiLanguageCode>
<TabItem value="js">

```js {3} title="stacks/index.js"
const authStack = new AuthStack(app, "auth");

new ApiStack(app, "api", authStack.auth);
```

</TabItem>
<TabItem value="ts">

```ts {3} title="stacks/index.ts"
const authStack = new AuthStack(app, "auth");

new ApiStack(app, "api", authStack.auth);
```

</TabItem>
</MultiLanguageCode>

Finally, attach the permissions.

<MultiLanguageCode>
<TabItem value="js">

```js {13} title="stacks/ApiStack.js"
import { Api, Stack } from "@serverless-stack/resources";

export class ApiStack extends Stack {
  constructor(scope, id, auth) {
    super(scope, id);

    const api = new Api(this, "Api", {
      routes: {
        "GET  /notes": "src/list.main",
        "POST /notes": "src/create.main",
      },
    });
    auth.attachPermissionsForAuthUsers([api]);
  }
}
```

</TabItem>
<TabItem value="ts">

```ts {13} title="stacks/ApiStack.ts"
import { Api, App, Auth, Stack } from "@serverless-stack/resources";

export class ApiStack extends Stack {
  constructor(scope: App, id: string, auth: Auth) {
    super(scope, id, props);

    const api = new Api(this, "Api", {
      routes: {
        "GET  /notes": "src/list.main",
        "POST /notes": "src/create.main",
      },
    });
    auth.attachPermissionsForAuthUsers([api]);
  }
}
```

</TabItem>
</MultiLanguageCode>

### Importing an existing User Pool

Override the internally created CDK `UserPool` and `UserPoolClient` instance.

```js {5-8}
import { UserPool, UserPoolClient } from "aws-cdk-lib/aws-cognito";

new Auth(this, "Auth", {
  cognito: {
    cdk: {
      userPool: UserPool.fromUserPoolId(this, "IUserPool", "pool-id"),
      userPoolClient: UserPoolClient.fromUserPoolClientId(this, "IUserPoolClient", "pool-client-id"),
    },
  },
});
```