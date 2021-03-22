# Porter Declarative Bundles

All Porter bundles are declarative. Mixins are v1 bundles with invocation images.

1. Determine what template values are created by a step.
2. Inject the Porter mixin after the step and have it modify a statefile in /porter. Re-evaluate remaining steps against statefile and save to step.yaml.
3. All steps that use templating get a step input file that the Porter mixin has modified as the parameter source.

```yaml
credentials:
- name: AZURE_CLIENT_ID
- name: AZURE_CLIENT_SECRET

install:
- a: createUser
  outputs:
  - name: username
- b: createDatabase # needs credentials: AZURE_CLIENT_ID, AZURE_CLIENT_SECRET
  credentials: # "all", "matching", or list
  - name: AZURE_CLIENT_ID
  - name: AZURE_CLIENT_SECRET
  arguments:
  - !t steps.a.username
  outputs:
  - name: connectionString
- a: deploy
  credentials:
  - name: connStr
    value: !t steps.createDatabase.outputs.connectionString
outputs:
- name: connectionString
```

Becomes

0. Mixin Config (saved at build-time to volume, allow multiple volumes to be mounted instead of just a single one)
  * /porter/mixins/a/config.yaml
  * /porter/mixins/b/config.yaml
  * /porter/steps/createUser/parameters/step.yaml (includes raw templating)
  * /porter/steps/createDatabase/parameters/step.yaml (includes raw templating)
1. porter (copy over any params and credentials used by the step that were provided by the bundle)
   -> /porter/steps/createDatabase/paramters/input.env (?)
   -> /porter/steps/createDatabase/credentials/AZURE_*
2. createUser
    <- /porter/steps/createUser/parameters/step.yaml (no templating at this point)
    -> /porter/steps/createUser/outputs/username.txt
3. porter applies username to the next step's input file
    -> /porter/steps/createDatabase/parameters/step.yaml
4. createDatabase
    <- /porter/steps/createDatabase/parameters/step.yaml (no templating at this point)
    -> /porter/steps/createDatabase/outputs/connectionString.txt 
    
5. porter
    -> /porter/steps/deploy/credentials/connectionString.txt
6. a
    <- /porter/steps/deploy/credentials/connectionString.txt
    <- /porter/steps/deploy/parameters/step.yaml
7. porter
    -> Copies bundle outputs to /cnab/outputs/

### Mixin Bundle Interface
Mixins have pre-defined files that they expect to be provided, in well-known locations that Porter will handle populating by the time the mixin step is called.

parameters:
- (envFile) input.env
- (file) step.yaml
- (file) config.yaml
credentials:
- no requirements, use anything
volume:
- /cnab
- /porter/currentStep (mounts just the /porter/steps/NAME for the current step)