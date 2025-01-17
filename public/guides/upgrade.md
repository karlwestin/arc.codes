# Architect upgrade guide

This document covers upgrading from previous versions of Architect.

As a general philosophy, Architect's core maintainers endeavor to minimize the frequency and impact of breaking changes wherever possible; in many cases, major releases may have no impact on existing applications.


## Overview of Architect versions

### Architect 4 (Yeti)

Architect 4 (Yeti) introduced generic, dependency-free HTTP functions, enhanced static asset support, and improved configurability. Information on upgrading from Yeti and versions prior to 4 are still available at the [Architect 5 docs archive](https://v5.arc.codes/guides/upgrade).


### Architect 5 (Catalope)

Architect 5 (Catalope) represented a major milestone in the project's functionality, introducing `@ws` (WebSocket) support. Catalope and was the last version to rely primarily on SDK calls to provision AWS infrastructure, and is currently in [LTS status](#architect-5).


### Architect 6 (Ogopogo)

Architect 6 (Ogopogo) was a ground-up rewrite of Architect, driven by user feedback, cloud vendor best practices, and extensive learnings by Architect maintainers from the first three years of the project's life.

Ogopogo is backward compatible with Architect 4 & 5, but depending on how you authored your project, there may have been [breaking changes going from 4/5 to 6](#architect-5-to-6).

What breaking changes there are, we have attempted to provide simple, forwards-compatible upgrade paths wherever possible.


### Architect 7 (Chupacabra)

Architect 7 (Chupacabra) evolves the Architect web application stack by defaulting to API Gateway `HTTP` APIs. (AWS considers `HTTP` APIs "v2.0"; the `REST` APIs provisioned by Architect since 2017 are now considered "v1.0".)

Deploying to an existing Architect project (that makes use of REST APIs) is completely forwards compatible; **no breaking infrastructure changes will be applied by Architect 7 unless manually and explicitly opted into**.

That said, Architect Sandbox workflows may potentially be impacted by this change. [See more below](#architect-6-to-7).

---

### Topics

<a href=#architect-6-to-7><b>Architect 6 &rarr; 7</b></a>

<a href=#architect-6-maintenance><b>Architect 6 maintenance</b></a>

<a href=#architect-5-to-6><b>Architect 5 &rarr; 6</b></a>

<a href=#architect-functions><b>Architect Functions runtime helper library (`@architect/functions`)</b></a>

<a href=#architect-data><b>Architect Data module (`@architect/data`)</b></a>

<a href=#architect-5><b>Architect 5 LTS maintenance schedule</b></a>

---

## <span id=architect-6-to-7>Architect 6 &rarr; 7</span>

### Overview

tldr – if you have an existing Architect 6 project:
- **You can continue to safely deploy that project with Architect 7**
- No breaking infrastructure changes will be applied by Architect 7 unless manually and explicitly opting in
- However, it is possible **Sandbox may be broken for your local workflows and testing**
  - If so, you'll need to **add a new setting API type setting** ([see breaking changes](#architect-7-breaking-changes))

We know the "`HTTP`" and "`REST`" API nomenclature is confusing – don't REST APIs use HTTP? Can't you use an HTTP API to build a REST interface? – but AWS named these API types, not us. Please allow us to do our best to explain!

AWS now offers two API Gateway types for marshaling (non-WebSocket) HTTPS requests and responses:
1. **`REST`** APIs - what Architect provisioned by default through version 6.x, aka "API Gateway v1.0"
2. **`HTTP`** APIs - what Architect provisions by default starting in version 7.0, aka "API Gateway v2.0"
  - Additionally, `HTTP` APIs integrate with Lambda using one of two request / response payload format versions:
    1. **Lambda payload format version 2.0** - the latest format, designed from the ground up for integrating `HTTP` APIs + Lambda
    2. **Lambda payload format version 1.0** – the legacy format, which provides close – *but not exact* – emulation of legacy `REST` API + Lambda payloads
    - [Learn more about `HTTP` API payload formats here](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html#http-api-develop-integrations-lambda.proxy-format)

Architect 7 (Chupacabra) evolves the Architect web application stack by **defaulting to API Gateway `HTTP` APIs using Lambda payload format version 2.0**.

`HTTP` APIs with Lambda payload format version 2.0 are themselves a breaking change with `REST` APIs; `HTTP` APIs with Lambda payload format version 1.0 are largely (but may not be 100%) compatible with `REST` APIs.

Architect 7 retains full backward compatibility for existing Architect 6 projects by continuing to deploy the same API type as you're currently using.


### Other changes

- New apps will default to using `HTTP` APIs, but can be configured as `REST` APIs – more on that below


### Removed

- Removed experimental support for static mocks
  - This was a very obscure experimental feature and should not impact anyone (but can be restored by building a macro)


### <span id=architect-7-breaking-changes>Breaking changes

All breaking changes in Architect 7 pertain to local development and testing with Sandbox, Architect's development environment. **But good news: Sandbox 2.0, included in Architect 7, also has compatibility paths for Architect 6 that only require a few seconds of your time.**

A core goal of Sandbox is to operate entirely locally and offline; that means no phoning home to AWS to introspect your current application configuration. Sandbox must run entirely from your machine using only your Architect project manifest. Thus, with the addition of `HTTP` APIs, Sandbox must now have a default API type.

It naturally follows that Sandbox's default would now be `HTTP` (with Lambda payload format version 2.0), since that is now the default for new projects. Because [API Gateway `HTTP` APIs introduced breaking changes with `REST` APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html#http-api-develop-integrations-lambda.proxy-format), existing projects may not function correctly until reconfigured to operate in `REST` mode.


#### Sandbox API type

The following are the valid settings for API types:
- **`http` (default)** - `HTTP` API + Lambda payload format version 2.0
  - `httpv2` – aliased to `http`
- **`httpv1`** - `HTTP` API + Lambda payload format version 1.0
  - Use this if you'd like to take advantage of `HTTP` APIs, but aren't yet ready to refactor your code for payload format 2.0
- **`rest` (previous default)** - `REST` API + original API Gateway payload format
  - Essentially the same as what is now called Lambda payload format version 1.0

Changing API types from `rest` to `http` **is a partially destructive change** that will result in new API Gateway URLs being generated. **Your old URLs will be destroyed** – if not accounted for, this may result in a service outage.

Once you're using a `HTTP` API, you can safely toggle between `http/httpv2` and `httpv1` non-destructively (although because the payloads differ, your code may no longer function; see below for information on seamlessly normalizing request / response payloads with `@architect/functions`).

#### Configuring the Sandbox API type

Configure your project's API type one of the following ways:

**1. Architect project manifest**

Using `httpv1` as your API type:
```arc
# app.arc|.arc|arc.yaml|etc.
@aws
apigateway httpv1
```

or:

**2. Environment variable**

Using `rest` as your API type via CLI:
```sh
ARC_API_TYPE=rest npx arc sandbox
```


#### Sandbox test environment changes

- Several seldom used and largely undocumented Sandbox module APIs intended for testing have breaking changes:
  - `sandbox.start()` no longer returns a function to shut down, and should now be shut down directly with `sandbox.end()`
  - `sandbox.db()` is now `sandbox.tables()`
  - `http.close()` is now `http.end()`
  - `events.start()` & `tables.start()` no longer return server objects to be invoked with `.close()`, and should now be shut down directly with `events.end()` and `tables.end()`
- Please see Sandbox for additional information:
  - [Sandbox 2.0 API](https://github.com/architect/sandbox/blob/master/readme.md)
  - [Breaking changes to undocumented module APIs](https://github.com/architect/sandbox/blob/master/changelog.md#200-2020-09-15)


### Upgrade scenarios

If your existing Architect 6 app **does not use `@http` or `@static` pragmas**, you're already ready to use to Architect 7.

If, like many, you use `@http` or `@static`, and you've configured Sandbox to operate `REST` API mode per the above instructions, you're also ready to use Architect 7.

However, you may also want to use Architect 7 to upgrade your existing `REST` API to the shiny `HTTP` stuff. If so, read on, but note: **is a partially destructive change** that will result in new API Gateway URLs being generated. **Your old URLs will be destroyed** – if not accounted for, this may result in a service outage.


#### Why upgrade to `HTTP` APIs?

If your Architect app is working well, you may not need to upgrade to `HTTP` APIs – in fact, depending on how you make use of your existing `REST` API, it may not be an upgrade at all – `HTTP` APIs are newer, and do not have some of the deeper, more obscure functionality of the older, more mature `REST` API system.

For most applications most of the time, we now believe `HTTP` APIs are the right API Gateway type to use, and there are some compelling reasons to upgrade to `HTTP` APIs:

- `HTTP` APIs are designed to be lower-latency
- `HTTP` APIs provision and integrate changes significantly faster
- `HTTP` APIs are significantly less expensive: as of this writing, they cost ≤$1.00/million requests, compared to `REST` APIs, which charge $3.50/million requests (plus data transferred)
- Default stages and routes, meaning we can finally escape the dreaded API Stage Path Part Problem (e.g. `/staging` in `https://{id}.execute-api.{region}.amazonaws.com/staging`)
- `HTTP` APIs are where AWS is now putting the bulk of its API Gateway development effort
- As of September 2020, `HTTP` APIs now support authorizers (which can be implemented via Architect Macros)
- For more information, please [compare `HTTP` to `REST` APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html), and see [Architect issue #838](https://github.com/architect/architect/issues/838)


#### Upgrading from `REST` to `HTTP`

As mentioned above: **this is a partially destructive change if performed on a running project**, and will result in a new API Gateway URLs being generated, and your old URLs being deactivated.

If you need a zero-downtime means of upgrading from `REST` to `HTTP`, the simplest and most reliable built-in means for doing so is to create a second app for your new `HTTP` API.

Creating a second app can be accomplished by deploying your existing app to a different AWS (org) account than your current app (i.e. by changing is keys), or simply by changing your app's name. Each has its own set of trade-offs and may result in additional complexity during your migration, so we encourage you to carefully consider and research the process by which you intend to migrate.

When you're ready to upgrade your existing Architect 6 (`REST`) app to `HTTP`, here's how:

**1. Architect project manifest**

Add the following to your project manifest:

```arc
# app.arc|.arc|arc.yaml|etc.
@aws
apigateway http
```

Then run a deploy:

```sh
npx arc deploy # Staging environment
npx arc deploy --production # Production environment
```

or:

**2. One-time deployment using the `--apigateway` flag via CLI**

```sh
npx arc deploy --apigateway http # Staging environment
npx arc deploy --apigateway http --production # Production environment
```

Tip: if you'd like to use `HTTP` APIs with code authored for an existing `REST` API project, manually specify the Lambda v1.0 payload format with `httpv1`.

As you might expect, backward compatibility for `REST` APIs is retained with the `rest` setting; should you need to revert to `REST` mode, apply that via CLI with `--apigateway rest`, or in project manifest with `@aws apigateway rest`. (Again, that will destroy your URLs and generate new ones, so plan ahead for that.)


#### Apps that use `@architect/macro-http-api` macro

Unfortunately, we have observed strange side effects when using the Architect 6 `@architect/macro-http-api` macro when deploying with Architect 7.

If you are currently using `@architect/macro-http-api` in production, please exercise caution when upgrading to Architect 7.

We've observed the following behavior in CloudFormation when running Architect 7 and `@architect/macro-http-api`:

- If you leave the `@architect/macro-http-api` macro in your app and deploy with Architect 7, CloudFormation will leave your existing API intact, and deploy a second API alongside it
  - This may be useful during a transition phase, but we cannot guarantee changes will be properly reflected in both APIs
- If you remove the `@architect/macro-http-api` macro from your app and deploy with Architect 7, CloudFormation will remove your original API (and its corresponding URL), leaving you with a fresh API and URL

Because CloudFormation behavior is subject to change, and due to the aforementioned observed side effects, we advise existing `@architect/macro-http-api` macro users exercise caution upgrading to Architect 7.

**If you are using `@architect/macro-http-api` in production today, you should not upgrade to Architect 7 until you've conducted field testing of your own, and are certain you are ready to transition your API URLs**.


### Compatibility with `@architect/functions`

If you're using `@architect/functions`, good news! `>= 3.13.0` is fully forward compatible with `HTTP` APIs and backward compatible with `REST` APIs. This means:
- You can use existing `REST` API code with `HTTP` APIs when run through `@architect/functions`
- You can implement `@architect/functions` in your codebase to ease any future `REST` to `HTTP` upgrades

Caveat: [per AWS](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html#http-api-develop-integrations-lambda.proxy-format), `HTTP` APIs + Lambda payload format version 2.0 does not have support in requests for `multiValueHeaders` or `multiValueQueryStringParameters`, so any code relying on those parameters should be adjusted, whether using `@architect/functions` or not.


### Additional resources

- [Learn more about `HTTP` API payload formats here](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html#http-api-develop-integrations-lambda.proxy-format)
- [Compare `HTTP` to `REST` APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)
- [Architect issue 838: `Promoting HTTP APIs to the default`](https://github.com/architect/architect/issues/838)

---

## <span id=architect-6-maintenance>Architect 6 maintenance</span>

Unlike [Architect 5, which remains in LTS (long-term support)](#architect-5), as of September 2020, Architect 6 will no longer be actively maintained. This is specifically because Architect 7 is a largely forward compatible release that only contains potentially breaking changes in Sandbox.

We suggest upgrading to Architect 7 as soon as possible to take advantage of the great new features planned for 7.x.

---

## <span id=architect-5-to-6>Architect 5 &rarr; 6</span>

### Overview

Architect 6 is a milestone release that solves some of the most crucial feedback we've received over the last two years. There's a lot to highlight below, but the first thing to know is that for some (but not all) Architect users, upgrading to Architect 6 may have breaking changes. We'll cover those below.


### About Architect 6

- **Architect 6 is now fully CloudFormation based**
  - This means your apps are provisioned and updated deterministically through AWS's first-party IaC (infrastructure as code) systems
  - This enables even tighter security profiles for Architect apps, and cleaner provisioning and teardown of infrastructure
  - As a minor example, if you remove a cloud function from your Architect project manifest, it'll no longer be left orphaned in your AWS app, requiring manual cleanup
  - Additionally, you'll no longer see resource creation rate limit errors, as that end of the infra provisioning logic is now handled entirely by AWS
- **Architect 6 now has first-class support for Ruby and Python**
  - This includes Ruby and Python support for the `sandbox` local dev server
  - It also includes ports of the Architect Functions module for [Ruby](https://github.com/architect/arc-functions-ruby) and [Python](https://github.com/architect/arc-functions-python)
    - Note: Functions for Ruby and Python is not yet at feature parity with the Node.js version, largely dictated by the availability and consistency of certain lower-level APIs in those runtimes.
    - As such, broadly speaking, if using Architect Functions in your [HTTP functions](/primitives/http) we recommend the Node.js version, which includes the most extensive support for front-end facing use cases.
- **Architect 6 now provisions CDNs with the `@cdn` pragma**
  - Finally, provision fast, fully-featured, global CDNs to live in front of your web app
  - This enables your web application to take advantage of crucial features like edge caching and global points of presence
- **Architect 6 is now a globally installable CLI tool, and is fully modular**
  - With Architect 6, you have the choice of installing Architect to your project (with Architect commands now prefixed by `npx arc`), or by installing it globally (`npm i -g @architect/architect`)
  - Architect is also now modular; for example, you can run the `sandbox` dev server in isolation (`npm i @architect/sandbox`) without needing all of Architect, or build custom workflows with other Architect modules like `@architect/deploy`


### Other changes

- Functions' dependencies are now automatically hydrated upon starting the `sandbox` dev server for the first time
- When creating new functions, runtime dependencies are fully opt-in
  - Unlike previous versions, Architect 6 no longer automatically creates a `package.json` with `@architect/functions` pre-installed
  - To install `@architect/functions` (or any other dependency) in a function, simply:
    1. Create a `package.json` file in that function dir (e.g. `touch src/events/a-background-task/package.json`)
    2. Add an empty `dependencies` object (see below)
    3. Then simply install whatever you please from within that directory (e.g. `cd src/events/a-background-task/ && npm i a-small-dependency`).
Example basic cloud function `package.json`:

```json
{
  "dependencies":{
    "a-small-dependency": "^1.0.0"
  }
}
```


### Removed

A number of Architect v5 workflows have since been deprecated, including:

- **`npx create`** - **Replaced**
  - To create boilerplate code (i.e. `npx create --local`) you should now use `npx arc init`
    - Or alternately, just start your project in `sandbox`, which will auto-initialize it)
  - Otherwise, create and deploy your app's live infra using `npx arc deploy`
- **`npx audit`** - **Removed**
  - App infra IAM roles are dynamically generated and scoped with least-privilege to various cloud infra services
- **`npx config`** - **Removed**
  - No longer necessary, as `.arc-config` settings are now serialized as part of every deploy
- **`npx inventory`** - **Removed**
  - Most prior uses of the `inventory` command are now addressed by the AWS CloudFormation console
- **`npx inventory --nuke`** - **Temporarily removed**
  - This command destroys your app infra, and is not currently available in Architect 6 (but will be returning shortly!)
  - [Follow this feature here](https://github.com/architect/architect/issues/430)
  - In the mean time, you can manually delete your app (aka "stack" in CloudFormation parlance) from the AWS CloudFormation console
- **`npx dns`** - **Temporarily removed**
  - This feature is not currently available in Architect 6 (but will be returning shortly!)
  - [Follow this feature here](https://github.com/architect/architect/issues/431)
  - In the mean time, you can manually add a domain to your app from the AWS console for API Gateway or CloudFront CDN (depending on how you've configured your Architect app to deliver HTTP)


### <span id=architect-6-breaking-changes>Breaking changes

- DynamoDB triggers (i.e. `@tables` `insert`, `update`, `destroy` Lambdas) are now defined with DynamoDB Streams enabled (`streamEnabled = true`)
  - [Learn more about DynamoDB Streams here](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_StreamSpecification.html)
- The dependency-free `request` and `response` signatures of Node.js-based HTTP functions have changed from the standard Architect format. These changes are non-breaking if you use `@architect/functions` – [more on this below](#upgrade-path).
- However, if you use dependency-free, `async/await` style Node.js HTTP functions, the following describes the specific breaking changes to the `request` and `response` signatures:


#### `request` breaking changes

The following Architect 5 `request` parameters changed in Architect 6:
- `method` is now `httpMethod`
  - Still one of `GET`, `POST`, `PATCH`, `PUT`, or `DELETE`
- `body` is no longer a pre-parsed object
  - `body` is now either `null` or a base64-encoded buffer
  - `body` must first be decoded, then parsed, to make use of it; Architect provides a handy helper to take care of this for you, see: [parsing HTTP request bodies](/primitives/http#parsing-request-bodies)
- `params` is now `pathParameters`
  - Still an object containing any URL params, if defined in your HTTP function's path (e.g. `foo` in `GET /:foo/bar`)
- `query` is now `queryStringParameters`
  - Still an object containing any query params if present in the client request
- Note: the `request` signature change is fully papered over with `@architect/functions`; if you are already using `@architect/functions` as your HTTP functions' handlers, you don't have to change anything, but you will need to upgrade to version `^3.3.0`


#### `response` breaking changes

The following Architect 5 `response` parameters changed in Architect 6:
- `location` is now deprecated
  - To set the `Location` of your content, please do so in the `headers` object, e.g. `headers: {'Location': '/new/path'}`
- `cookie` is now deprecated
  - `cookie` is now respected interchangeably with `session` by `@architect/functions`, however
- `status` & `code` are now `statusCode`
  - Still a `number` that sets the HTTP status code
  - Note: Arc 5 also supported `statusCode`, so if you're using that, there's no effective change
- `type` is now deprecated
  - To set the `Content-Type` of your content, please do so in the `headers` object, e.g. `headers: {'Content-Type': 'text/html; charset=utf-8;'}`


### Upgrade path

Architect support two styles of authoring Node.js cloud function handlers:
- Continuation-passing style (i.e. `callback`s), which uses Architect Functions to deliver a `response` (in addition to other features, such as middleware, etc.)
- `async/await` style, which does not depend on Architect Functions to deliver a `response`

The upgrade path for each of which is covered below:


#### Continuation-passing style HTTP functions (via `arc.http`)

If your HTTP functions are authored continuation-passing style using Architect Functions (i.e. `arc.http`), you have no breaking code changes. Simply run `npx arc hydrate --update` and ensure all your HTTP functions are running `@architect/functions` version `^3.3.0` (or greater).


#### <span id=arc-http-async>`async/await` style HTTP functions (via `arc.http.async`)</span>

If you already use `@architect/functions`'s `arc.middleware` (now `arc.http.async`) with your functions, you have no breaking code changes. Simply run `npx arc hydrate --update` and ensure all your HTTP functions are running `@architect/functions` version `^3.3.4` (or greater).

If your HTTP functions are authored in `async/await` style without `arc.http.async`, you'll have two paths forward to ensure compatibility with Architect 6:

1. Continue to opt out of using `@architect/functions` with your functions
- If you opt not to use `@architect/functions`, per the [list of signature changes above](#architect-6-breaking-changes), you'll need to update any logic related to `request`s and `response`s to the new Architect 6 signatures; generally this should be fairly low-impact and straightforward, but your mileage may vary

2. Opting to use `@architect/functions` with your functions
- You can opt into using `@architect/functions` with a very minor code change, namely by running your existing functions through `arc.http.async`, like so:


#### Example (before)
```javascript
// `async/await` Arc 5 function is incompatible with Arc 6
exports.handler = async function handler(request) {
  let name = request.body.email   // Accessor will fail, as `request.body` is no longer automatically parsed
  return {
    status: 200,                  // Response will fail, `status` param no longer valid
    type: 'text/html'             // Response will fail, `type` param no longer valid
    body: `<h1>Hi ${name}</h1>`
  }
}
```

#### Example (after)
```javascript
// Same `async/await` Arc 5 function made Arc 6 compatible via `arc.http.async`
let arc = require('@architect/functions')

exports.handler = arc.http.async(handler)

async function handler(request) {
  let name = request.body.email   // `request.body` automatically parsed by arc.http.async
  return {
    status: 200,                  // `status` param valid when passed through arc.http.async
    type: 'text/html'             // `type` param valid when passed through arc.http.async
    body: `<h1>Hi ${name}</h1>`
  }
}
```

---

## <span id=architect-functions>Architect Functions module (`@architect/functions`)</span>

### Overview

`@architect/functions` will continue to be actively maintained, and is forwards compatible with Architect 6 &rarr; 7 upgrades, and most Architect 5 &rarr; 6 upgrades.

The Architect Functions module is now also available as a dependency for [Ruby](https://github.com/architect/arc-functions-ruby) and [Python](https://github.com/architect/arc-functions-python) functions.


### Changes

- `arc.http.middleware` was deprecated in August 2019, and is now `arc.http.async`
  - These methods are functionally the same
- `arc.http.helpers.static` was deprecated in June 2019, and is now `arc.static`
  - These methods are functionally the same
  - Due to some under-the-hood changes, if you use `arc.http.helpers.static` or `arc.static`, you will need to upgrade to `@architect/functions` version `^3.3.4` (or greater) in Architect 6
- `arc.proxy` was deprecated in May 2020, and is now `arc.http.proxy`
  - These methods are functionally the same

In all three cases, these are functionally the same. The old aliases will remain for a while to come, but we suggest moving any deprecated calls over to their new equivalents by mid-2020.


### Does `@architect/functions` work in Architect 7?

Yes! It is supported by and forwards compatible in Architect 7, including use with HTTP API Lambda v2.0 payloads.


### Does `@architect/functions` work in Architect 6?

Yes! It is supported by and forwards compatible in Architect 6. Additionally, it has been expanded to include [`@tables` support for working with data](/reference/functions/tables).


### Will `@architect/functions` continue working in Architect 5?

Yes! `@architect/functions` is fully backward compatible with Architect 5. You can safely update this dependency, and expect related bugs to be patched. **However, it is worth noting that the new `tables()` method is an Architect 6-only feature.**

---

## <span id=architect-data>Architect Data module (`@architect/data`)</span>

As of the release of Architect 6, the **Architect Data module (`@architect/data`) is now deprecated**, and has been superseded by `@architect/functions` `tables()` method.


### Overview

`@architect/data` will no longer be maintained, and upgrading to Architect 6 will likely be a breaking change.


### Does `@architect/data` work in Architect 6?
No, this module was deprecated with the release of Architect 6.


### Will `@architect/data` continue working in Architect 5?
Yes. The lower-level AWS APIs within `@architect/data` are not expected to change in the near future, so indefinite use of `@architect/data` should be considered generally safe **if you choose not to upgrade to Architect 6**. That said, we strongly suggest you upgrade to receive the latest fixes, features, and updates.


### Upgrade path
If you built your `@architect/data` calls with `async/await`, `@architect/functions` `tables()` is largely a drop-in replacement. However, **if you are using callbacks, `@architect/functions` `tables()` is a breaking change.**


### Example of `@architect/data` &rarr; `@architect/functions` `tables()`

#### `@architect/data` before:
```js
// src/http/get-index/index.js
let data = require('@architect/data')

exports.handler = async () => {
  let body = await data.accounts.get({key:'foo'})
  return {body}
}
```


#### `@architect/functions` `tables()` after:
```js
// src/http/get-index/index.js
let {tables} = require('@architect/functions')

exports.handler = async () => {
  let data = await tables()
  let body = await data.accounts.get({key:'foo'})
  return {body}
}
```

---

## <span id=architect-5>Architect 5 LTS maintenance schedule</span>

As of the release of Architect 6 in August 2019, Architect 5 is in LTS (long-term support). Architect maintainers will fix CVEs / security issues and critical bugs for a minimum of 6 months (i.e. through February 2020), and we expect that Architect 5 will continue working more or less hassle-free for some time to come.

Non-critical bugs, feature requests, PR merges, etc. may be completed on a discretionary and case-by-case basis.
