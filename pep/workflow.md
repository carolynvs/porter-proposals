OCI Index
 - manifest
 - layers
    - bundle files (volume)
    - porter (invocation image)
    - exec mixin image
    - helm mixin image

Pull Bundle
* Retrieves the bundle and referenced images
* Populates docker cache
* On k8s that's once per node (can use https://github.com/senthilrch/kube-fledged to help cache)

Mount bundle info into invocation image
* Create invocation image
* Mount data volume to /cnab/app
* Mount referenced images to /cnab/images
* Copy bundle.json to /cnab/bundle.json

What if the porter operator delegated execution to argo/brigade/whatever?
- tool collects parameters and credentials (outter bundle inputs)
- plugin translates the desired porter workflow (intermediate representation of DAG) to something the executor understands
- the tool creates a container workflow based on the workflow plugin
  - porter operator would execute an argo workflow
  - porter generates an internal container based workflow

```toml
default-workflow = "my-brig" # defaults to porter.workflow.docker

[[workflow]]
  name = "my-brig"

  [workflow.config]
    kubeconfig = ~/dev-briagde.kubeconfig
```