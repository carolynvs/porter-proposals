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

What if the porter operator delegated execution to argo?
- tool collects parameters and credentials (outter bundle inputs)
- the tool creates a container workflow based on the workflow plugin
  - porter operator would execute an argo workflow
  - porter generates a 
- 