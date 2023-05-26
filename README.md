# Scaleway plugin for Serverless Framework

Plugin for using Scaleway [Serverless Functions](https://www.scaleway.com/en/serverless-functions/) and [Serverless Containers](https://www.scaleway.com/en/serverless-containers/) with [Serverless Framework](https://serverless.com/).

**Requirements**:

- You have installed the [Serverless Framework](https://serverless.com) CLI on your local computer (to do so, run `npm install serverless -g` in a terminal).
- You have set up the [Scaleway CLI](https://www.scaleway.com/en/cli/)

If you are using [Scaleway IAM](/identity-and-access-management/iam/how-to/activate-iam), you need to be the [Owner](/identity-and-access-management/iam/concepts/#owner) of the Scaleway [Organization](/identity-and-access-management/iam/concepts/#organization) in which the actions will be carried out, or you are an IAM user of the Organization, with a [policy](/identity-and-access-management/iam/concepts/#policy) granting you the necessary [permission sets](/identity-and-access-management/iam/reference-content/permission-sets/).

## Quickstart

1. Export the template you wish to use (see the list [here](https://github.com/scaleway/serverless-scaleway-functions/tree/master/examples)). We will use `python3`:

  ```shell
  export TEMPLATE=python3
  ```

2. Create a function from this template:

  ```shell
  serverless create --path my-func --template-url https://github.com/scaleway/serverless-scaleway-functions/tree/master/examples/${TEMPLATE}
  ```

3. Install dependencies:

  ```shell
  cd my-func
  npm i
  ```

4. Deploy the function. The URL is returned:

  ```shell
  serverless deploy
  ```

5. Invoke the function (note that `first` is the function name in this example):

  ```shell
  serverless invoke --function first
  ```

## Contents

Serverless Framework handles everything from creating namespaces to function/code deployment by calling API endpoints under the hood.

- [Quickstart](#quickstart)
- [Configuration](#configuration)
- [Supported commands](#supported-commands)
- [Security and secret management](#security-and-secret-management)
- [Events](#events)
- [Custom domains](#custom-domains)
- [Deployment methods](#deployment-methods)
- [Local testing](#local-testing)
- [Managing containers](#managing-containers)
- [Logs](#logs)
- [Info](#info)
- [Documentation and useful links](#documentation-and-useful-links)
- [Contributing](#contributing)
- [License](#license)

## Configuration

Your functions are defined in the `serverless.yml` file.

Each `serverless.yml` file corresponds to one function _or_ container namespace.

The file format is described below:

```yml
# The name of your namespace
service: scaleway-python3

# Read environment variables from a .env file
useDotenv: false

# Use of this plugin. This must not be changed
plugins:
  - serverless-scaleway-functions

# Scaleway-specific configuration
provider:
  # Must not change
  name: scaleway

  # Function runtime
  # List: https://www.scaleway.com/en/docs/serverless/functions/reference-content/functions-lifecycle/#available-runtimes
  runtime: python310

  # Global environment variables, used in every function
  env:
    MY_VAR: "some var"
    MY_OTHER_VAR: "some other var"

  # Global secrets, used in every function
  secret:
    MY_SECRET: "some secret"
    MY_OTHER_SECRET: "some other secret"

  # Optional override of Scaleway credentials
  scwToken: <scw-token>
  scwProject: <scw-project-id>

  # Scaleway region for the deploy
  scwRegion: fr-par

# Include/exclude directories
package:
  patterns:
    - '!node_modules/**'
    - '!.gitignore'
    - '!.git/**'

# Define functions - cannot be used with custom.containers
functions:
  my-func:
    # Handler entrypoint
    handler: handler.py

    # Minimum and maximum number of instances of the function
    minScale: 0
    maxScale: 10

    # Memory limit for the function (in MiB)
    # Limits: https://www.scaleway.com/en/docs/serverless/functions/reference-content/functions-limitations/
    memoryLimit: 1024

    # Runtime for this function, allows overriding provider.runtime
    runtime: node20

    # Whether to permit HTTP or HTTPS-only. Options: enabled, disabled, or redirected
    httpOption: enabled

    # Whether the container is public (no authentication), or private (uses token-based authentication)
    privacy: public

    # Local environment variables, used only in this function
    env:
      LOCAL_VAR: "local var"

    # Local secrets, used only in this function
    secret:
      LOCAL_SECRET: "local secret"

    # Custom domains configured for the function
    # https://www.scaleway.com/en/docs/compute/functions/how-to/add-a-custom-domain-name-to-a-function/
    custom_domains:
      - my-func.some.domain.com

    # List of events to trigger the function
    events:
      - schedule:
          rate: '1 * * * *'
          # Data passed as input in the request
          input:
            key-a: "value-a"
            key-b: "value-b"

# Define containers, cannot be used with functions
custom:
  containers:
    my-container:
      # Subdirectory holding the Dockerfile, cannot be used with registryImage
      directory: container/

      # Name of the registry image, cannot be used with directory
      registryImage: nginx:latest

      # Minimum and maximum number of instances of the container
      minScale: 0
      maxScale: 10

      # Number of simultaneous requests to handle simultaneously
      maxConcurrency: 20

      # Memory limit for the container, in MiB
      # Limits: https://www.scaleway.com/en/docs/serverless/containers/reference-content/containers-limitations/
      memoryLimit: 1024

      # CPU limit for the container in mvCPU, chosen based on resource tiers if not set
      # Limits and tiers: https://www.scaleway.com/en/docs/serverless/containers/reference-content/containers-limitations/
      cpuLimit: 1000

      # Maximum duration a request will wait to be served before it times out (in seconds)
      timeout: 300

      # Whether to permit HTTP or HTTPS-only. Options: enabled, disabled, or redirected
      httpOption: enabled

      # Whether the container is public (no authentication), or private (uses token-based authentication)
      privacy: public

      # Local environment variables, used only in this container
      env:
        LOCAL_VAR: "local var"

      # Local secrets, used only in this container
      secret:
        LOCAL_SECRET: "local secret"

      # Custom domains configured for the function
      # https://www.scaleway.com/en/docs/serverless/containers/how-to/add-a-custom-domain-to-a-container/
      custom_domains:
        - my-container.some.domain.com

      # List of events to trigger the container
      events:
        - schedule:
            rate: '1 * * * *'
            # Data passed as input in the request
            input:
              key-a: "value-a"
              key-b: "value-b"
```

### Deployment methods

At Scaleway, there are multiple ways to create Serverless Functions and Serverless Containers. These include: the CLI, APIs, the Scaleway console, Serverless Framework and Terraform.

The `serverless deploy` command applies the configuration located in your `serverless.yml` and removes functions that are not in the file to ensure a single source of truth.

This can be controlled using the `singleSource` option. By default its value is `false`.

If `singleSource` is set to `true`, functions and containers not defined in your serverless configuration file will be removed the next time you run the `serverless deploy` command.

### Local testing

`serverless invoke local` is **not supported** directly but instead we provide additional packages to install close to your handler.

Documentation is available through runtimes frameworks for :
* [Go](https://github.com/scaleway/serverless-functions-go)
* [Python](https://github.com/scaleway/serverless-functions-python)
* [Node](https://github.com/scaleway/serverless-functions-node)

### Managing containers

>**Requirements:**
- You have [created a Container Registry namespace](https://www.scaleway.com/en/docs/compute/container-registry/how-to/create-namespace/)
- You have installed Docker and can build and push your image to your registry.

To manage your containers, you must first define them in the `custom.containers` field in your `serverless.yml` configuration file.

Each container must specify the relative path of its application directory (containing the Dockerfile, and all files related to the application to deploy):

```yml
custom:
  containers:
    mycontainer:
      directory: my-container-directory
      # port: 8080
      # Environment only available in this container
      env:
        MY_VARIABLE: "my-value"
```

Below is an example of the files you should have in your application directory. The directory that contains your Dockerfile and scripts is called `my-container-directory`.

```
.
├── my-container-directory
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── server.py
│   └── (...)
├── node_modules
│   ├── serverless-scaleway-functions
│   └── (...)
├── package-lock.json
├── package.json
└── serverless.yml
```

Scaleway's platform will automatically inject a PORT environment variable on which your server should be listening for incoming traffic. By default, this PORT is 8080. You can change the `port` in the `serverless.yml` file.

You can use the container example provided on this [documentation page](https://github.com/scaleway/serverless-scaleway-functions/tree/master/examples/container) to get started.

## Logs

The `serverless logs` command lets you watch the logs of a specific function or container.

You can fetch the logs of a specific function for with the `--function` option. You must specify the name of your function in the command.

```bash
serverless logs --function <function_or_container_name>
```

## Info

The `serverless info` command gives you information about your functions' or containers' current deployement state in JSON format.

## Documentation and useful links

- [Official Scaleway Serverless Functions Documentation](https://www.scaleway.com/en/docs/compute/functions/api-cli/fun-uploading-with-serverless-framework/)
- [Official Scaleway Serverless Containers Documentation](https://www.scaleway.com/en/docs/compute/containers/api-cli/cont-uploading-with-serverless-framework/)
- [Serverless Framework documentation](https://serverless.com)
- [Scaleway Cloud Provider](https://scaleway.com)
- [Scaleway Serverless sample projects](https://github.com/scaleway/serverless-examples)

## Troubleshooting

### Rate Limiting Issue

If you are experiencing rate limiting issues (error 429) in your application, consider engaging with the support and/or the community. When seeking assistance, remember to provide relevant details, such as the specific rate limiting error messages, the affected components, and any relevant configuration information. This will enable us to better understand your situation and provide appropriate guidance or solutions.

## Contributing

This plugin is developed and maintained by the `Scaleway Serverless Team`, but we welcome pull requests and issues, and are available to chat on our [Community Slack Channels](https://scaleway-community.slack.com/): #serverless-containers and #serverless-functions.

For general information about developing Serverless Framework, refer to the Serverless Framework [plugins documentation](https://www.serverless.com/framework/docs/guides/plugins/creating-plugins).

To run Serverless Framework with a local checkout of this plugin, you can modify the `serverless.yml` for one or more functions as follows:

```yaml
...

# Change this
plugins:
  - serverless-scaleway-functions

# To this
plugins:
  - <path to checkout of this project>
```

Then you can run commands as normal.

## License

This project is MIT licensed.
