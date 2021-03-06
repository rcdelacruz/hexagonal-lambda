# Hexagonal Lambda

This is an example application repository for implementing [hexagonal architecture](http://alistair.cockburn.us/Hexagonal+architecture) as described by Alistair Cockburn (also called "ports and adapters") on top of Amazon Web Services Lambda.

It is intended to be a reference/example project implementation. While small, it is hopefully realistic and reasonably comprehensive. It takes into account development tooling, testing, deployment, security, developer documentation, and API documentation.

## Setting up for development the first time

- Install prerequisites
  - node and npm
    - See `.nvmrc` file for correct node version
    - Using [https://github.com/creationix/nvm](nvm) recommended but optional
  - zip
  - pass: `brew install pass`
  - AWS CLI:  `brew install awscli`
    - **OR** `virtualenv python && ./python/bin/pip install awscli`
- Clone the git repo if you haven't already and `cd` into the root directory
- Run `npm install && npm run lint && npm test`
- Set up your `local/env.sh` based on the template below. More details on configuration further down in this document.

```
export AWS_DEFAULT_REGION='us-example-1'
export PASS_ENV='hexagonal-lambda-dev'
export HL_DEPLOY='dev'
export HL_HTTPBIN_URL='https://httpbin.org'
export TF_VAR_httpbin_url="${HL_HTTPBIN_URL}"
```
- To use that for terminal development we do `source ./local/env.sh`;

- Set up pgp and a private key for working with `pass`
  - There's some docs missing here on initially setting up a PGP key if you've never had one before and initializing your password store/repo. I'm currently pondering different alternatives for this so bear with me while I figure that out. Start with `gpg --full-generate-key`.
- Set up the following shell alias (run in project root directory):
  - `alias run-pass="${PWD}/bin/run-pass.sh"`

## How to do typical development

- Build lambdas: `npm run build`
- Run lint: `npm run lint`
- Run tests: `npm test`
  - Run a single test file: `NODE_ENV=test tap code/get-hex/unit-tests-tap.js`
  - Run some tests in a  file: `NODE_ENV=test tap --grep=example code/get-hex/unit-tests-tap.js`
    - This will only run tests whose description includes "example"
  - Run a few test files: `NODE_ENV=test tap code/get-hex/unit-tests-tap.js code/post-up/unit-tests-tap.js`
  - debug a single test file: `NODE_ENV=test node --inspect-brk --inspect code/get-hex/unit-tests-tap.js`
    - Or since debugging in node v6.10 which is what AWS lambda supports at the moment is buggy, you can debug in newer node:
    - `NODE_ENV=test nvm exec v8.9.0 node --inspect --inspect-brk code/get-hex/unit-tests-tap.js`
- Run a lambda locally: `node code/get-hex/run.js`
  - Edit the sample event object in the code as needed before calling the lambda handler to simulate a specific case of interest
- Run code coverage: `npm run coverage`
- Preview terraform: `(cd terraform/dev && run-pass terraform plan)`
- Provision for real: `(cd terraform/dev && run-pass terraform apply)`
- Run smoke (integration/system) tests
  - `export HL_API_URL=$(cd terraform/dev && run-pass terraform output api_url)`
    - where "terraform/dev" is the desired deployment to test
  - `npm run smoke` with the appropriate values for your deployment
- Trigger an API Gateway deployment: `run-pass npm run deploy-apig dev`
  - Substitute `demo` for `dev` to target that deployment
  - Note our terraform-triggered deployments currently have an ordering issue where they fire before other APIG changes are done, so manually deploying is sometimes required.
- Build OpenAPI JSON for documentation: `run-pass npm run openapi`
  - Spits out JSON to stdout. Copy/paste to a swagger UI if you want a pretty site.
  - `run-pass npm run openapi demo` if you want to set the demo deployment as the base URL
- Update secrets
  - for dev: `(cd secrets/dev && PASSWORD_STORE_DIR=. pass edit secrets.sh)`
  - for demo same put replace dev with demo
- Import
## Filesystem Layout

This project follows the same [underlying principles](https://github.com/focusaurus/express_code_structure#underlying-principles-and-motivations) I describe in my "Express Code Structure" sample project. Terraform doesn't play well with this as it requires grouping all `.tf` files in the same directory, so those are in a separate directory.

**File Naming Conventions**

- `lamba.js` AWS lambda handler modules
- `*-tap.js` tap unit test files
- `smoke-tests.js` smoke test files
- `openapi.js` Open API documentation as an object

## Lambda Organization

- Each lambda handler goes in its own directory under `code/name-of-lambda`
  - This directory contains
  - The lambda code itself goes in `lambda.js`
    - The handler function is exported as `exports.handler`
- I use the `mintsauce` middleware npm package to allow me to concisely mix and match reusable middlewares across all my lambdas
  - The middleware pattern from express is proven effective, but it has drawbacks around implicit middleware interdependencies and run order
  - Try not to over-rely on `call.local` shared state
- All lambda code is easy to test and develop on
  - Lambda tests are fully runnable offline
  - Lambda handlers are fully runnable local talking to AWS/Internet services
  - The whole test suite is fast to run
  - It's easy to run a single test file
  - It's easy to run a small group of test files
  - It's easy to run some or all tests under the devtools debugger
  - It's easy to run a lambda handler under the devtools debugger
- Each lambda has a corresponding `-tap.js` file for the unit tests
- Each lambda has a corresponding `terraform/*.tf` that defines the terraform configuration for that lambda function, and a corresponding API Gateway method as needed

## Input Validation

All key data shapes including end user input and external service responses is modeled as JSON schema and validated immediately upon arrival into the system. JSON schema has broad tool support (OpenAPI, many npm packages, etc) and is also cross-language. Currently we use `ajv` for validation as it is quite thorough and the error messages are good, although it's API is awkward. For each data shape, we have examples easily available for unit tests and ad-hoc developer testing convenience.

The `code/core/schemas.js` module provides some helper functions to make JSON schema easier to work with including `check(input)` and `example()` helper functions as properties on the schema object itself.

## Lambda Error Reporting

For lambdas triggered by API Gateway, most errors are "soft errors" and should be done via `callback(null, res);` where `res.statusCode` is the appropriate HTTP 400/500 value. I only pass an error as the first callback argument for programmer/deployment errors that will require developer/admin attention to fix. Examples would be invalid lambda environment variables or IAM errors accessing AWS resources. But an external service failing, invalid end user input, anything that might resolve itself with time should be considered "success" from the lambda callback perspective.

## Configuration

Non-secret configuration can be set as environment variables prefixed with `HL_` to distinguish them from the other variables present in your environment. Use the `local/env.sh` file to configure them into your shell. This file is excluded from git to allow each developer the ability to set distinct personal settings if desired.

Like external input, configuration data is considered external and thus we define the expected schema in JSON schema and validate it as early as possible and refuse to process invalid configuration. The code takes configuration key/value string settings from environment variables (both for local development and when running in lambda), and validates the configuration is sufficient before using that data.

When tests are run (`NODE_ENV=test`) a realistic but neutered/harmless ("example.com" etc) test configuration is forceably set so the test environment is consistent.

## Credentials and Secrets

Developers will be working with a set of AWS credentials when interacting with AWS APIs. These are security sensitive and must remain confidential and thus there's a fairly byzantine system described here about how we try to secure them. We generally don't want them available to be stolen by shoulder surfing or unauthorized access to your laptop or stolen by malicious install scripts during `npm install`. Thus the are encrypted at rest using the [pass](https://www.passwordstore.org/) password storage system. This essentially makes a database of secrets encrypted via [OpenPGP](https://en.wikipedia.org/wiki/Pretty_Good_Privacy#OpenPGP) and associated tools such as gnupg, gpg-agent, etc.

When running commands interacting with AWS such as `aws` or `terraform`, we'll use `./bin/run-pass.sh` in combination with gpg-agent to decrypt your permanent AWS credentials, which are then used with the AWS session token service to generate a set of temporary credentials, which we then expose to the subcommand actually doing the work as environment variables in a single short-lived subprocess.

Initial setup of secrets was done basically as follows:

```
cat <<EOF | pass -m insert hexagonal-lambda-dev
export AWS_ACCESS_KEY_ID='EXAMPLE'
export AWS_SECRET_ACCESS_KEY='example-secret-access-key'
export AWS_MFA_SERIAL='arn:aws:iam::1111111111:mfa/example-username'
export AWS_PROFILE='example'

export HL_AWS_ACCOUNT='1111111111'
export HL_SECRET1='example-secret-1'

export TF_VAR_aws_account="${HL_AWS_ACCOUNT}"
export TF_VAR_hl_secret1="${HL_SECRET1}"
EOF
```

Once you have a snippet of bash script stored in your password store, you can use the `run-pass` shell alias described above to to expose temporary credentials plus the application's secrets to subcommands.

**Debugging run-pass.sh** can be enabled by `export RUN_PASS_DEBUG=yes` in your shell. Disable with `unset RUN_PASS_DEBUG`.

## API Documentation

The docs are defined as a javascript object matching the OpenAPI v2 spec. This gets printed to JSON. It's tiny. Just read it as JSON but if you must you can paste it into a [Swagger UI Editor](https://editor.swagger.io) if you like.

The basic structure and shared things are defined in `bin/build-openapi.js` and then each endpoint's `openapi.js` file merges in the openapi data for it's own endpoint path.

## Hexagonal Architecture: External Services

Any external service dependencies are represented with a clean function call boundary. The API is designed to be a clean port/adapter to a third party system. When testing our main internal code, we mock the external services and synthesize responses/errors as necessary to fully test our main code. This generally simplifies the potential responses to just success or failure, with all failures represented with standard node errback pattern.

For testing the glue code that interacts with the remote service, we mock HTTP responses with nock to make sure our adapter code handles all cases properly.

We design our AWS event-driven architecture to make sure the scope/responsibility of any particular lambda function is small enough to fully test without a huge amount of backing service mocking ceremony. Rule of thumb would be if you have to make 3 service calls start looking for a way to split into several lambdas by using kinesis, dynamodb streams, etc.

## Logging

I keep it very simple as I've found the following sufficient so far. Use `null-console` in tests to silence the logs and otherwise just use good old `console` for the logging you do. The middleware for automatically logging the input event is handy, just use caution if you expect any secrets or personal information in there and obfuscate those before logging.

## Provisioning with Terraform

There is terraform configuration here to provision all the necessary AWS resources. It works OK but it can be a bit boilerplatey and tedious.

Why not AWS CloudFormation? Mostly because terraform is multi-cloud and therefore more broadly valuable to know, and frankly because I heard at AWS re:Invent 2016 the community had mostly switched to terraform over cloudformation.

### DynamoDB Initial Setup

```sh
for deploy in global dev demo;
do
  aws dynamodb create-table \
    --table-name "hl-${deploy}-terraform" \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1
done
```
## Smoke Tests

My approach to end-to-end system testing I think is best described as "smoke testing" in that I run a small/minimal set of tests that ensure me the target environment is basically operational (not billowing out smoke). These smoke tests do not exhaustively test the application though.  I rely on my unit tests to thoroughly test the code's functionality. The smoke tests run quickly due to their very limited scope and are generally harmless and/or read-only operations.

They can be run against different environments by setting the `API_URL` environment variable to the API Gateway URL including the path prefix corresponding to the deployment stage but no trailing slash.

## Tooling

For lambda builds at the moment I am using **webpack** and terraform for deployments. I haven't found a better way of managing depnedencies and getting small zip files. Webpack allows me to

- Share 1 package.json file for all lambdas in the repo, which keeps things simple
- Only bundle the production dependencies, no dev/test dependencies
- Only bundle the specific dependencies of each individual lambda function. Because webpack walks the `require` dependency graph, each lambda gets only what it actually uses. (This is only to package granularity, no tree shaking to get even more precise than that)
  - The `get-hex.zip` file is well under 1 MB, for example
  - In theory the single-file bundling might even speed up the lambda cold startup time due to fewer filesystem reads, but that's just maybe a tiny side benefit
- When I looked at serverless framework in Oct 2016 it was still pretty clunky and didn't seem to have a viable dependency management solution. That may have already been satisfactorily fixed or if not might be soon. I plan to re-evaluate serverless as it matures and adopt if/when it becomes useful.
- I like the comprehensiveness that terraform can provision almost every resource I need, both the "serverless" ones and most other random AWS stuff as well.
- I think learning/using terraform is more broadly useful/valuable vs just knowing serverless.

I use **eslint** for static analysis. Very valuable.

I use **prettier** for automatic code formatting. No configuration. Great wrapping of long lines.

I prefer **tap** for my test runner because the API doesn't require much nesting of functions, the matching API is memorable and effective, and code coverage tooling is integrated out of the box.

Thus when I need credentials for an `aws` command or terraform, etc, I run it like so:

`run-pass aws dynamodb list-tables`

This exposes the AWS credentials to that one command only via environment variables. aws-vault also handles MFA/STS properly.

## Style vs Substance

I don't care so much about this particular set of npm dependencies, this particular testing stack, this code formatting style, etc. It's more about the substance:

- We are confident the code is correct and robust
  - Hitting a surprise bug after deployment is a rare event
- We can simulate base/happy cases as well as edge/error cases in tests
  - Otherwise there's no way to know what will happen that 1 time a year when that S3 HTTP GET fails
- We can make changes effeciently and offline
  - Rapid cycle of edit/test (near-zero delay)
- We are protected against basic issues
  - eslint config handles a lot of basic typos, etc
  - near-100% code coverage means code was actually executed locally before being committed, pushed, or deployed
- Developers can navigate the code easily
  - Easy to find which file to edit
  - Easy to edit all files necessary to make a typical change
