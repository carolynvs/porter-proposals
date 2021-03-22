# Declarative Bundles
* The bundle's invocation image is only a filesystem, it doesn't have an entry point and is never invoked.
* Declares "components" which are bundles, and they are executed when the bundle is run.
* They share the same claim.r
* They can reference other component's outputs as their parameters and credentials.
* The bundle manifest contains all of the component bundles.

## What is the difference between a bundle component and a bundle dependency?
* components are resolved at build time, dependencies are resolved at runtime
* components are included in the bundle manifest, dependencies are not.
* if a component has a dependency, it becomes a dependency of the including bundle and is resolved at runtime.
* dependencies of bundle components store data on the bundle's installation, cannot be managed separately
* dependencies of bundles get their own installation
* bundle components are stateless. They have stateless: true. They do not store state, outputs are not persisted.
* use a bundle component to manipulate the invocation image and include tools, common logic
* use dependencies to manage resources independently of the bundle

## Communication with Bundle Component
* via shared invocation image filesystem
* parameters
* outputs

### How does a component generate env vars to pass into other components?
1. Add new parameter type for a file with env vars in it. 
    ```yaml
    parameters:
    - name: porter.env
      type: envFile # Does this need to be in the spec? It would be nice to have file parameter types as a custom extension.
      path: /cnab/app/porter.env
      # prefix: AZURE_
    ```
1. The runtime will create env vars based on the file contents
    ```
    PORTER_DEBUG=true
    VERBOSE=10
    ```

## Is there still a porter runtime with this?

* Templating
    What if porter replaced all template values with a file on the invocation image. Then add a custom extension for file parameters, and use it.
* Output manipulation (creating outputs by scraping stdout, files, etc) runs in the mixin's bundle. We create an output directory and save any outputs there for later use with steps.STEP.outputs.OUTPUT

## How is a mixin translated into a bundle call?

Example with Parameters and Outputs
```yaml
- exec:
    name: cfg-connstr
    command: ./helpers.sh # The script has access to /cnab/app/kube.config and $MYSQL_USERNAME
    arguments:
      - !t {{ steps.wordpress.dependencies.mysql.outputs.host }} # Can access outputs from a step dependency without declaring them
      - !t {{ steps.wordpress.outputs.user }} # Don't use bundle.outputs anymore, reference by step 
    outputs: # Allow mixins to generate undeclared outputs
      - name: scapeStuff
        regex: "(.*)"
```

```yaml
- bundle:
    name: cfg-connstr
    reference: getporter/mixin-exec
    parameters:
      - name: config # what if we stopped doing stdin and just wrote these to files?
        value: <insert mixin config> # /cnab/app/mixin-config.yaml
      - name: step # includes the outputs declaration
        value: <insert step yaml> # /cnab/app/mixin-step-input.yaml
      - TODO: magic to render the arguments templated values
```

Example with Credentials

```yaml
- az:
    name: login
    arguments:
      - login
    credentials: 
      - name: AZURE_TOKEN
```

```yaml
- bundle:
    name: create-storage-account
    reference: getporter/mixin-az
    version: v0.11.2
    parameters:
      - name: config
      - name: step-input
    credentials:
      - name: AZURE_TOKEN
        source:
          credential: AZURE_TOKEN
```

Pass all matching credentials
```yaml
- az:
    name: login
    arguments:
      - login
    matchCredentials: true
```

```yaml
- bundle:
    name: create-storage-account
    reference: getporter/mixin-az
    version: v0.11.2
    parameters:
      - name: config
      - name: step-input
    credentials:
      - name: AZURE_TOKEN
        source:
          credential: AZURE_TOKEN
```

## Sample porter.yaml
This uses the new templating custom yaml tag, !t, to indicate that the node's value is templated with handlebars.

```yaml
name: mybuns
version: !t bundle.custom.appVersion

# variables defined at build time
custom:
  appVersion: v1.2.3

# keywords used to search and inspect a bundle
keywords: 
 - asciiart

# labels for the bundle, applied to any installations
labels:
  app: myservice

parameters:
- name: release-name
  default: !t {{ bundle.parameters.user }}-wordpress-{{ generateName 7}} # generates carolynvs-wordpress-abc1234
  # the default is stored as "" in bundle.json, porter defaults it when the bundle is executed using porter-runtime
  # This makes the bundle compatible with other tools without adding more to the spec

state: # when present load into bundle, may not have been generated yet
- name: tfstate
  path: terraform/tfstate
  # required
  # applyTo

install:
  - bundle:
      description: Create Cluster
      reference: getporter/aks:v0.2.0
      name: aks
      outputs:
        - name: kubeconfig # Rename an output for use in this bundle
          source: user-kubeconfig # use the name when source is omitted
          path: kube.config # store the output at this location on the invocation image
  - bundle: # the wordpress installation is stored with in this bundle's claim, it can't be managed separately
      description: Configure Wordpress
      reference: getporter/wordpress-cfg:v0.1.0
      name: wordpress
      parameters:
        installationName: !t {{ bundle.parameters.wordpressName }}
        kubeconfig: !t {{ steps.aks.outputs.kubeconfig }} # if something is of type file, expose the path not the contents
      outputs: # Mixins always expose to other steps their outputs, the author can add more
        - name: user # Rename an output from a dependency and expose it on the bundle
          source: dependencies.mysql.outputs.username # parameters.databaseName
          env: MYSQL_USERNAME
  - exec:
      command: ./helpers.sh # The script has access to /cnab/app/kube.config and $MYSQL_USERNAME
      arguments:
        - !t {{ steps.wordpress.dependencies.mysql.outputs.host }} # Can access outputs from a step dependency without declaring them
        - !t {{ steps.wordpress.outputs.user }} # Don't use bundle.outputs anymore, reference by step
      
  - az:
      description: Setup DNS
      matchCredentials: true # Pass in any matching credentials without having to wire each one up
      arguments:
        - dns
        - add
        - www.example.com
        - !t {{ steps.wordpress.outputs.ip }}
        - !t {{ bundle.parameters.output }} # force a string with "", if no quotes we try to strcon.ParseInt and ParseBool

upgrade:
   - bundle: # what do we do with dependencies from not-install?
       description: bundle not used by install 
       name: roguebuns
       reference: getporter/rogue-one:v2.0.0
       action: # defaults to the action of the current bundle, upgrade. it's okay to skip install because it gets to use the current bundle's installation.

dependencies: # when dependencies can reference other dependencies should that affect the execution order? we need to at least call it out if we aren't being smart
  - name: redis
    reference: getporter/redis
    version: v1.x
  - name: mymicroservice
    reference: my/microservice
    version: !t {{ bundle.custom.appVersion }}
    parameters:
      redisEndpoint: !t {{ bundle.dependencies.redis.outputs.url }}
```

## what does the bundle.json and oci manifest look like?

```json
{
    "custom": {
        "io.cnab.dependencies": {
            "redis": {},
            "mymicroservice": {
                "parameters": {
                    "redisEndpoint": {
                        "sources": {
                            "priority": ["output", "default"],
                            "output": {
                                "name": "url",
                                "dependency": "redis"
                            },
                            "default": "redis.example.com"
                        }
                    }
                }
            }
        }
    } 
}
```

# Spec Changes
* invocation image is a volume,  not an executed container
* We can mount multiple volumes (/cnab, /porter)
* output can be a directory. we can output things that weren't declared
* envFile parameter type and file parameter type (allows for larger files)
* 