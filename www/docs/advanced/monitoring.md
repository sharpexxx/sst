---
title: Monitoring
description: "Learn how to use services like Datadog, Sentry, Epsagon, and Lumigo to monitor the Lambda functions in your SST app in production."
---

import config from "../../config";
import HeadlineText from "@site/src/components/HeadlineText";
import MultiPackagerCode from "@site/src/components/MultiPackagerCode";

<HeadlineText>

SST integrates with all the major observability providers to help you monitor your apps in production.

</HeadlineText>

Once your app has been [deployed to production](../going-to-production.md), there are a few different options on how to monitor your Lambda functions. Let's look at them here.

---

## Datadog

[Datadog](https://www.datadoghq.com) offers an [End-to-end Serverless Monitoring](https://www.datadoghq.com/product/serverless-monitoring/) solution that works with Lambda functions. It consists of the following three distinct libraries:

1. [`datadog-cdk-constructs-v2`](https://github.com/Datadog/datadog-cdk-constructs) — a CDK construct that sets up Lambda layers and environment variables. It also provides a Lambda handler override that keeps the Datadog API key in an AWS Secrets Manager Secret. This is highly recommended, although technically optional, as you could add the Lambda layers, environment variables, and the Datadog API key separately through SST if you prefer.
2. [`dd-trace`](https://github.com/Datadog/dd-trace-js) — Datadog's JavaScript APM tracer. This is necessary if you wish to add traces to your Lambda functions.
3. [`datadog-lambda-js`](https://github.com/Datadog/datadog-lambda-js) — a Lambda-specific version of `dd-trace`. This is necessary if you wish to add traces or custom metrics to your Lambda functions.

Start by adding these to your project:

<MultiPackagerCode>
<TabItem value="npm">

```bash
npm install --save-dev datadog-cdk-constructs-v2 dd-trace datadog-lambda-js
```

</TabItem>
<TabItem value="yarn">

```bash
yarn add --dev datadog-cdk-constructs-v2 dd-trace datadog-lambda-js
```

</TabItem>
<TabItem value="pnpm">

```bash
pnpm add --save-dev datadog-cdk-constructs-v2 dd-trace datadog-lambda-js
```

</TabItem>
</MultiPackagerCode>

:::note
These are added as dev dependencies. The `datadog-cdk-constructs-v2` package is used by SST, not by your Lambda code. The `dd-trace` and `datadog-lambda-js` packages are not be bundled with your Lambda functions, but rather be loaded from the Lambda layer provided by Datadog's CDK construct.
:::

When you setup an AWS account to be monitored by Datadog, the process uses a CloudFormation stack to set up an API key for forwarding logs, metrics, traces, and other observability data to your Datadog account. The CloudFormation stack sets up this API key in an AWS Secrets Manager Secret, and it can be re-used by the Datadog serverless monitoring CDK construct.

To monitor all the functions in an app, add the following to your `sst.config.ts`:

```ts title="sst.config.ts"
import {SSTConfig} from "sst";
import {Stack} from "sst/constructs";
import {Datadog} from "datadog-cdk-constructs-v2";
import {Effect, PolicyStatement} from "aws-cdk-lib/aws-iam";

export default {
  config() {
    // Config
  },
  async stacks(app) {
    // Replace with the API Key Secret ARN from your
    // Datadog Integration CloudFormation stack
    const datadogApiKeySecretArn =
    "arn:aws:secretsmanager:us-east-1:123456789012:secret:DdApiKeySecret-abcdef123456-789ABC";

    // Don't enable locally
    const enableDatadog = !app.local;

    if (enableDatadog) {
      // Allow functions to access secret
      app.addDefaultFunctionPermissions([
        new PolicyStatement({
          effect: Effect.ALLOW,
          resources: [datadogApiKeySecretArn]
          actions: ["secretmanager:GetSecretValue"],
        })
      ]);
      // Exclude from the function bundle
      // since they'll be loaded from the Layer
      app.setDefaultFunctionProps({
        nodejs: {
          install: ["dd-trace", "datadog-lambda-js"],
        },
      });
    }

    // Add your stacks
    app.stack( /* ... */ );

    if (enableDatadog) {
      await app.finish();

      // Attach the Datadog contruct to each stack
      app.node.children.forEach((stack) => {
        if (stack instanceof Stack) {
          const datadog = new Datadog(stack, "datadog", {
            // Get the latest version from
            // https://github.com/Datadog/datadog-lambda-js/releases
            nodeLayerVersion: 87,
            // Get the latest version from
            // https://github.com/Datadog/datadog-lambda-extension/releases
            extensionLayerVersion: 40,
            site: "datadoghq.com",
            apiKeySecretArn: datadogApiKeySecretArn,
            env: app.stage,
            service: app.name,
            // Just a recommendation, feel free to change the version per your CI/CD
            version:
              process.env.SEED_BUILD_SERVICE_SHA ||
              process.env.GITHUB_SHA ||
              undefined,
          });

          datadog.addLambdaFunctions(stack.getAllFunctions());
        }
      });
    }
  }
} satisfies SSTConfig;
```

Then in each Lambda function, should wrap it with the handler provided by `datadog-lambda-js`:

```ts title="packages/functions/src/lambda.ts" {3}
import { datadog } from "datadog-lambda-js";

export const handler = datadog(async () => {
  return {
    statusCode: 200,
    body: "Hello World!",
  };
});
```

For more details, [check out the Datadog docs](https://docs.datadoghq.com/serverless/installation/nodejs/?tab=awscdk).

---

## Sentry

[Sentry](https://sentry.io) offers [Serverless Error Monitoring](https://sentry.io/for/serverless/) for your Lambda functions. Integration is done through a Lambda Layer.

Head over to the [Layer that Sentry provides](https://docs.sentry.io/platforms/node/guides/aws-lambda/layer/), select your region and copy the layer ARN.

Then add the Layer to your stack.

```ts title="stacks/Foo.js"
import { LayerVersion } from "aws-cdk-lib/aws-lambda";

const sentry = LayerVersion.fromLayerVersionArn(
  stack,
  "SentryLayer",
  `arn:aws:lambda:${scope.region}:943013980633:layer:SentryNodeServerlessSDK:34`
);
```

You can then set it for all the functions in your stack using the [`addDefaultFunctionLayers`](constructs/Stack.md#adddefaultfunctionlayers) and [`addDefaultFunctionEnv`](constructs/Stack.md#adddefaultfunctionenv). Note we only want to enable this when the function is deployed, not when using [Live Lambda Dev](live-lambda-development.md).

```ts title="stacks/Foo.js"
if (!scope.local) {
  stack.addDefaultFunctionLayers([layer]);
  stack.addDefaultFunctionEnv({
    SENTRY_DSN: "<SENTRY_DSN>",
    NODE_OPTIONS: "-r @sentry/serverless/dist/awslambda-auto",
  });
}
```

Sentry also offers performance monitoring for serverless. To enable, add the `SENTRY_TRACES_SAMPLE_RATE` environment variable.

```js {3}
stack.addDefaultFunctionEnv({
  SENTRY_DSN: "<SENTRY_DSN>",
  SENTRY_TRACES_SAMPLE_RATE: "1.0",
  NODE_OPTIONS: "-r @sentry/serverless/dist/awslambda-auto",
});
```

This can be tuned between the values of 0 and 1. Where 0 means that no performance related information is sent, and 1 means that information for all the invocations are sent. This should be tuned based on the volume of invocations and the amount of transactions available in your Sentry account. A value of 0.5 should work for most projects.

You also need to wrap your function handlers.

```js title="packages/functions/src/foo.js"
import * as Sentry from "@sentry/serverless";

export const handler = Sentry.AWSLambda.wrapHandler(async (event) => {
  // ...
});
```

For more details, [check out the Sentry docs](https://docs.sentry.io/platforms/node/guides/aws-lambda/).

---

## Lumigo

[Lumigo](https://lumigo.io) offers a [Serverless Monitoring and Debugging Platform](https://lumigo.io/).

To get started, [sign up for an account](https://platform.lumigo.io/signup). Then [follow their wizard](https://platform.lumigo.io/wizard) to deploy their stack in your AWS production account.

Then to enable Lambda monitoring for a function, add a `lumigo:auto-trace` tag and set it to `true`.

```js title="stacks/Foo.js"
import * as cdk from "aws-cdk-lib";

cdk.Tags.of(myfunc).add("lumigo:auto-trace", "true");
```

To monitor all the functions in a stack, you can use the [Stack](constructs/Stack.md) construct's [`getAllFunctions`](constructs/Stack.md#getallfunctions) method and do the following at the bottom of your stack definition.

```js title="stacks/Foo.js"
import * as cdk from "aws-cdk-lib";

stack
  .getAllFunctions()
  .forEach((fn) => cdk.Tags.of(fn).add("lumigo:auto-trace", "true"));
```

For more details, [check out the Lumigo docs on auto-tracing](https://docs.lumigo.io/docs/auto-instrumentation#auto-tracing-with-aws-tags).

---

## Thundra

[Thundra](https://thundra.io) offers [Thundra APM - Application Performance Monitoring for Serverless and Containers](https://thundra.io/apm).

To get started, [sign up for an account](https://console.thundra.io/landing/). Then [follow the steps in the quick start guide](https://apm.docs.thundra.io/getting-started/quick-start-guide/connect-thundra) to deploy their stack into the AWS account you wish to monitor.

To enable Lambda monitoring, you'll need to add a layer to the functions you want to monitor. To figure out the layer ARN for the latest version, [check the badge here](https://apm.docs.thundra.io/node.js/nodejs-integration-options).

With the layer ARN, you can use the layer construct in your CDK code.

```ts title="stacks/Foo.js"
import { LayerVersion } from "aws-cdk-lib/aws-lambda";

const thundraLayer = LayerVersion.fromLayerVersionArn(
  stack,
  "ThundraLayer",
  "<ARN>"
);
```

You can then set it for all the functions in your stack using the [`addDefaultFunctionLayers`](constructs/Stack.md#adddefaultfunctionlayers) and [`addDefaultFunctionEnv`](constructs/Stack.md#adddefaultfunctionenv). Note we only want to enable this when the function is deployed, not in [Live Lambda Dev](live-lambda-development.md) as the layer will prevent the debugger from connecting.

```ts title="stacks/Foo.js"
if (!scope.local) {
  const thundraAWSAccountNo = 269863060030;
  const thundraNodeLayerVersion = 94; // Latest version at time of writing
  const thundraLayer = LayerVersion.fromLayerVersionArn(
    stack,
    "ThundraLayer",
    `arn:aws:lambda:${scope.region}:${thundraAWSAccountNo}:layer:thundra-lambda-node-layer:${thundraNodeLayerVersion}`
  );
  stack.addDefaultFunctionLayers([thundraLayer]);

  stack.addDefaultFunctionEnv({
    THUNDRA_APIKEY: process.env.THUNDRA_API_KEY,
    NODE_OPTIONS: "-r @thundra/core/dist/bootstrap/lambda",
  });
}
```

For more details, [check out the Thundra docs](https://apm.docs.thundra.io/node.js/nodejs-integration-options).

---

#### Time Travel Debugging

Thudra also offers a feature called [Time Travel Debugging (TTD)](https://apm.docs.thundra.io/debugging/offline-debugging) that makes it possible to travel back in time to previous states of your application by getting a snapshot of when each line is executed. You can step over each line of the code and track the values of the variables captured during execution.

To enable TTD in your SST app, you'll need to modify the esbuild config. [Check out the Thundra docs on this](https://apm.docs.thundra.io/node.js/ttd-time-travel-debugging-for-nodejs#using-with-sst).

---

## New Relic

[New Relic](https://newrelic.com/) offers [New Relic Serverless for AWS Lambda](https://newrelic.com/products/serverless-aws-lambda).

To get started, [follow the steps in the documentation](https://docs.newrelic.com/docs/serverless-function-monitoring/aws-lambda-monitoring/get-started/monitoring-aws-lambda-serverless-monitoring/).

To enable Lambda monitoring, you'll need to add a layer to the functions you want to monitor. To figure out the layer ARN for the latest version, [check the available layers per region here](https://layers.newrelic-external.com/).

With the layer ARN, you can use the layer construct in your CDK code. To ensure the Lambda function is instrumented correctly, the function handler must be set to the handler provided by the New Relic layer. Note we only want to enable this when the function is deployed, not in [Live Lambda Dev](live-lambda-development.md) as the layer will prevent the debugger from connecting.

Add the following at the bootom of the `main()` function in your `stacks/index.ts` file.

```ts title="stacks/index.ts"
import { CfnFunction, LayerVersion } from "aws-cdk-lib/aws-lambda";

if (!app.local) {
  const runDeferredBuildsBk = app.runDeferredBuilds;
  app.runDeferredBuilds = async () => {
    await runDeferredBuildsBk();

    // Loop through each stack in the app
    app.node.children.forEach((stack) => {
      if (stack instanceof sst.Stack) {
        const newRelicLayer = LayerVersion.fromLayerVersionArn(
          stack,
          "NewRelicLayer",
          "<ARN>>"
        );

        child.getAllFunctions().forEach((fn) => {
          const cfnFunction = fn.node.defaultChild as CfnFunction;
          if (cfnFunction.handler) {
            fn.addEnvironment("NEW_RELIC_LAMBDA_HANDLER", cfnFunction.handler);
          }

          cfnFunction.handler = "newrelic-lambda-wrapper.handler";
        });
      }
    });
  };
}
```

---

## Epsagon

:::caution

Epsagon is undergoing some changes after the acquisition by Cisco. We recommend using one of the other monitoring services.

:::

[Epsagon](https://epsagon.com) is an end-to-end [Application Monitoring Service](https://epsagon.com/) and can monitor the full lifecycle of your serverless requests.

The Epsagon docs on [using a Lambda Layer](https://docs.epsagon.com/docs/getting-started/monitoring-applications/aws-lambda-layer) are incorrect. You'll need to install the Epsagon agent for your Lambda functions.

<MultiPackagerCode>
<TabItem value="npm">

```bash
npm install --save-dev epsagon
```

</TabItem>
<TabItem value="yarn">

```bash
yarn add --dev epsagon
```

</TabItem>
<TabItem value="pnpm">

```bash
pnpm add --save-dev epsagon
```

</TabItem>
</MultiPackagerCode>

And wrap your Lambda functions with their tracing wrapper.

```js title="packages/functions/src/foo.js"
const handler = epsagon.lambdaWrapper(function (event, context) {
  // Lambda code
});

export { handler };
```
