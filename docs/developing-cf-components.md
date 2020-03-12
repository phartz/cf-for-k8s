# Developing CF Components

## Additional Dependencies
In addition to the core build dependencies: [/docs/development.md#dependencies](/docs/development.md#dependencies), the workflow herein uses these tools:

- [kbld](https://get-kbld.io/)
- [minikube](https://github.com/kubernetes/minikube)

## Component Repository Organization
To simplify the component development workflow, we recommend repositories organize configuration thusly:

```
├── example-component
│   ├── config               - K8s resources
│   │   ├── deployment.yml
│   │   └── values
│   │       ├── _default.yml - contains schema/defaults for all values used within config
│   │       └── images.yml   - contains resolved image references (ideally in digest form)
│   └── build                - configuration for the build of CF-for-K8s
│       └── kbld.yml
```

_Note: a working example structure this way can be found at [`./example-component`](example-component)._

Notes:
- place all K8s configuration under one directory. e.g. `config/` _(so that while invoking `ytt` you always specify just that one directory.)_
- within the `config/` directory, gather data value file(s) into a sub-directory: e.g. `config/values/` _(so that while invoking `vendir sync` you always specify just that one directory.)_
- place anything required to configure building/generating K8s resource templates or data files in a separate directory. e.g. `build` _(so that all that's in the `config` directory are only the K8s resources being contributed to CF-for-K8s)_

## Potential Workflow

1. Create or claim a Kubernetes cluster.  We expect these instructions to work for any distro of cluster, your mileage may vary (local or remote).
1. Checkout `cf-for-k8s` develop and install it to your cluster.
1. Start a local Docker Daemon so that you can build (and push) local Docker images.

   Here, we install `minikube` for its Docker Daemon (and not necessarily as our target cluster).
    ```
    minikube start
    eval $(minikube docker-env)
    docker login -u ...
    ```
1. Make changes to your component
1. Rebuild your component image and update any image references in your configuration.

   Here, we use `kbld` and generate a ytt data value file (i.e. `config/values/images.yml`) with that resolved image reference:
    ```
    cd ~/workspace/example-component
    cat <(echo "#@data/values"; kbld -f build/kbld.yml --images-annotation=false) \
        >config/values/images.yml
    ```
    _Note: `ytt` merges data files in alphabetical order of the full pathname.  So, `config/values/_default.yml` is used first, THEN `config/values/images.yml` (see [k14s/ytt/ytt-data-values](https://github.com/k14s/ytt/blob/master/docs/ytt-data-values.md#splitting-data-values-into-multiple-files)).  Therefore, it is critical that whatever name you choose for the generated data file (here, `images.yml`), it sorts _after_ `_default.yml`._

    _Note: see [Sample kbld config](#sample-kbld-config), below for an example of a `kbld` config._

1. Sync your local component configuration into the `cf-for-k8s` repo.

   Here, we use vendir's [`--directory`](https://github.com/k14s/vendir/blob/985506a54038f6e7871879d4fbee9df2b6cf8add/docs/README.md#sync-with-local-changes-override) feature to sync _just_ the directory containing our component:

    ```
    cd ~/workspace/cf-for-k8s
    vendir sync \
      --directory config/_ytt_lib/<path-to-component>=~/workspace/<path-to-component>
    ```

    For example, sync'ing _just_ CAPI would look like this:

    ```
    cd ~/workspace/cf-for-k8s
    vendir sync \
      --directory config/_ytt_lib/github.com/cloudfoundry/capi-k8s-release=~/workspace/capi-k8s-release/
    ```
1. Re-deploy `cf-for-k8s` with your updated component.


### Sample kbld config

See [kbld docs](https://github.com/k14s/kbld/blob/master/docs/config.md) to configure your own `kbld.yml`.

Assuming your `ytt` template takes the data value at `image.example` as the image reference...

```yaml
images:
  example: example-component-image
---
apiVersion: kbld.k14s.io/v1alpha1
kind: Sources
sources:
- image: example-component-image
  path: .
---
apiVersion: kbld.k14s.io/v1alpha1
kind: ImageDestinations
destinations:
- image: example-component-image
  newImage: docker.io/<your-dockerhub-username>/<your-docker-repo-name>
---
apiVersion: kbld.k14s.io/v1alpha1
kind: ImageKeys
keys:
- example
```

Where:
- the first YAML document is a "template" into which `kbld` will rewrite the image reference.
- the remaining YAML documents are configuration for `kbld`, itself.
