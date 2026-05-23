
# Config Management Plugins

Argo CD's "native" config management tools are Helm, Jsonnet, and Kustomize. If you want to use a different config
management tool, or if Argo CD's native tool support does not include a feature you need, you might need to turn to
a Config Management Plugin (CMP).

The Argo CD "repo server" component is in charge of building Kubernetes manifests based on some source files from a
Helm, OCI, or Git repository. When a config management plugin is correctly configured, the repo server may delegate the
task of building manifests to the plugin.

The following sections will describe how to create, install, and use plugins. Check out the
[example plugins](https://github.com/argoproj/argo-cd/tree/master/examples/plugins) for additional guidance.

> [!WARNING]
> Plugins are granted a level of trust in the Argo CD system, so it is important to implement plugins securely. Argo
> CD administrators should only install plugins from trusted sources, and they should audit plugins to weigh their
> particular risks and benefits.

## Installing a config management plugin

### Sidecar plugin

An operator can configure a plugin tool via a sidecar to repo-server. The following changes are required to configure a new plugin:

#### Write the plugin configuration file

Plugins will be configured via a ConfigManagementPlugin manifest located inside the plugin container.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  # The name of the plugin must be unique within a given Argo CD instance.
  name: my-plugin
spec:
  # The version of your plugin. Optional. If specified, the Application's spec.source.plugin.name field
  # must be <plugin name>-<plugin version>.
  version: v1.0
  # The init command runs in the Application source directory at the beginning of each manifest generation. The init
  # command can output anything. A non-zero status code will fail manifest generation.
  init:
    # Init always happens immediately before generate, but its output is not treated as manifests.
    # This is a good place to, for example, download chart dependencies.
    command: [sh]
    args: [-c, 'echo "Initializing..."']
  # The generate command runs in the Application source directory each time manifests are generated. Standard output
  # must be ONLY valid Kubernetes Objects in either YAML or JSON. A non-zero exit code will fail manifest generation.
  # To write log messages from the command, write them to stderr, it will always be displayed.
  # Error output will be sent to the UI, so avoid printing sensitive information (such as secrets).
  generate:
    command: [sh, -c]
    args:
      - |
        echo "{\"kind\": \"ConfigMap\", \"apiVersion\": \"v1\", \"metadata\": { \"name\": \"$ARGOCD_APP_NAME\", \"namespace\": \"$ARGOCD_APP_NAMESPACE\", \"annotations\": {\"Foo\": \"$ARGOCD_ENV_FOO\", \"KubeVersion\": \"$KUBE_VERSION\", \"KubeApiVersion\": \"$KUBE_API_VERSIONS\",\"Bar\": \"baz\"}}}"
  # The discovery config is applied to a repository. If every configured discovery tool matches, then the plugin may be
  # used to generate manifests for Applications using the repository. If the discovery config is omitted then the plugin 
  # will not match any application but can still be invoked explicitly by specifying the plugin name in the app spec. 
  # Only one of fileName, find.glob, or find.command should be specified. If multiple are specified then only the 
  # first (in that order) is evaluated.
  discover:
    # fileName is a glob pattern (https://pkg.go.dev/path/filepath#Glob) that is applied to the Application's source 
    # directory. If there is a match, this plugin may be used for the Application.
    fileName: "./subdir/s*.yaml"
    find:
      # This does the same thing as fileName, but it supports double-star (nested directory) glob patterns.
      glob: "**/Chart.yaml"
      # The find command runs in the repository's root directory. To match, it must exit with status code 0 _and_ 
      # produce non-empty output to standard out.
      command: [sh, -c, find . -name env.yaml]
  # The parameters config describes what parameters the UI should display for an Application. It is up to the user to
  # actually set parameters in the Application manifest (in spec.source.plugin.parameters). The announcements _only_
  # inform the "Parameters" tab in the App Details page of the UI.
  parameters:
    # Static parameter announcements are sent to the UI for _all_ Applications handled by this plugin.
    # Think of the `string`, `array`, and `map` values set here as "defaults". It is up to the plugin author to make 
    # sure that these default values actually reflect the plugin's behavior if the user doesn't explicitly set different
    # values for those parameters.
    static:
      - name: string-param
        title: Description of the string param
        tooltip: Tooltip shown when the user hovers the
        # If this field is set, the UI will indicate to the user that they must set the value.
        required: false
        # itemType tells the UI how to present the parameter's value (or, for arrays and maps, values). Default is
        # "string". Examples of other types which may be supported in the future are "boolean" or "number".
        # Even if the itemType is not "string", the parameter value from the Application spec will be sent to the plugin
        # as a string. It's up to the plugin to do the appropriate conversion.
        itemType: ""
        # collectionType describes what type of value this parameter accepts (string, array, or map) and allows the UI
        # to present a form to match that type. Default is "string". This field must be present for non-string types.
        # It will not be inferred from the presence of an `array` or `map` field.
        collectionType: ""
        # This field communicates the parameter's default value to the UI. Setting this field is optional.
        string: default-string-value
      # All the fields above besides "string" apply to both the array and map type parameter announcements.
      - name: array-param
        # This field communicates the parameter's default value to the UI. Setting this field is optional.
        array: [default, items]
        collectionType: array
      - name: map-param
        # This field communicates the parameter's default value to the UI. Setting this field is optional.
        map:
          some: value
        collectionType: map
    # Dynamic parameter announcements are announcements specific to an Application handled by this plugin. For example,
    # the values for a Helm chart's values.yaml file could be sent as parameter announcements.
    dynamic:
      # The command is run in an Application's source directory. Standard output must be JSON matching the schema of the
      # static parameter announcements list.
      command: [echo, '[{"name": "example-param", "string": "default-string-value"}]']

  # If set to `true` then the plugin receives repository files with original file mode. Dangerous since the repository
  # might have executable files. Set to true only if you trust the CMP plugin authors.
  preserveFileMode: false

  # If set to `true` then the plugin can retrieve git credentials from the reposerver during generate. Plugin authors 
  # should ensure these credentials are appropriately protected during execution
  provideGitCreds: false
```

> [!NOTE]
> While the ConfigManagementPlugin _looks like_ a Kubernetes object, it is not actually a custom resource. 
> It only follows kubernetes-style spec conventions.

The `generate` command must print a valid Kubernetes YAML or JSON object stream to stdout. Both `init` and `generate` commands are executed inside the application source directory.

The `discover.fileName` is used as [glob](https://pkg.go.dev/path/filepath#Glob) pattern to determine whether an
application repository is supported by the plugin or not. 

```yaml
  discover:
    find:
      command: [sh, -c, find . -name env.yaml]
```

If `discover.fileName` is not provided, the `discover.find.command` is executed in order to determine whether an
application repository is supported by the plugin or not. The `find` command should return a non-error exit code
and produce output to stdout when the application source type is supported.

#### Place the plugin configuration file in the sidecar

Argo CD expects the plugin configuration file to be located at `/home/argocd/cmp-server/config/plugin.yaml` in the sidecar.

If you use a custom image for the sidecar, you can add the file directly to that image.

```dockerfile
WORKDIR /home/argocd/cmp-server/config/
COPY plugin.yaml ./
```

If you use a stock image for the sidecar or would rather maintain the plugin configuration in a ConfigMap, just nest the
plugin config file in a ConfigMap under the `plugin.yaml` key and mount the ConfigMap in the sidecar (see next section).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-plugin-config
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: my-plugin
    spec:
      version: v1.0
      init:
        command: [sh, -c, 'echo "Initializing..."']
      generate:
        command: [sh, -c, 'echo "{\"kind\": \"ConfigMap\", \"apiVersion\": \"v1\", \"metadata\": { \"name\": \"$ARGOCD_APP_NAME\", \"namespace\": \"$ARGOCD_APP_NAMESPACE\", \"annotations\": {\"Foo\": \"$ARGOCD_ENV_FOO\", \"KubeVersion\": \"$KUBE_VERSION\", \"KubeApiVersion\": \"$KUBE_API_VERSIONS\",\"Bar\": \"baz\"}}}"']
      discover:
        fileName: "./subdir/s*.yaml"
```

#### Register the plugin sidecar

To install a plugin, patch argocd-repo-server to run the plugin container as a sidecar, with argocd-cmp-server as its 
entrypoint. You can use either off-the-shelf or custom-built plugin image as sidecar image. For example:

```yaml
containers:
- name: my-plugin
  command: [/var/run/argocd/argocd-cmp-server] # Entrypoint should be Argo CD lightweight CMP server i.e. argocd-cmp-server
  image: ubuntu # This can be off-the-shelf or custom-built image
  securityContext:
    runAsNonRoot: true
    runAsUser: 999
  volumeMounts:
    - mountPath: /var/run/argocd
      name: var-files
    - mountPath: /home/argocd/cmp-server/plugins
      name: plugins
    # Remove this volumeMount if you've chosen to bake the config file into the sidecar image.
    - mountPath: /home/argocd/cmp-server/config/plugin.yaml
      subPath: plugin.yaml
      name: my-plugin-config
    # Starting with v2.4, do NOT mount the same tmp volume as the repo-server container. The filesystem separation helps 
    # mitigate path traversal attacks.
    - mountPath: /tmp
      name: cmp-tmp
volumes:
- configMap:
    name: my-plugin-config
  name: my-plugin-config
- emptyDir: {}
  name: cmp-tmp
``` 

> [!IMPORTANT]
> **Double-check these items**
>
> 1. Make sure to use `/var/run/argocd/argocd-cmp-server` as an entrypoint. The `argocd-cmp-server` is a lightweight GRPC service that allows Argo CD to interact with the plugin.
> 2. Make sure that sidecar container is running as user 999.
> 3. Make sure that plugin configuration file is present at `/home/argocd/cmp-server/config/plugin.yaml`. It can either be volume mapped via configmap or baked into image.

### Using environment variables in your plugin

Plugin commands have access to

1. The system environment variables of the sidecar
2. [Standard build environment variables](../user-guide/build-environment.md)
3. Variables in the Application spec (References to system and build variables will get interpolated in the variables' values):

        apiVersion: argoproj.io/v1alpha1
        kind: Application
        spec:
          source:
            plugin:
              env:
                - name: FOO
                  value: bar
                - name: REV
                  value: test-$ARGOCD_APP_REVISION
    
    Before reaching the `init.command`, `generate.command`, and `discover.find.command` commands, Argo CD prefixes all 
    user-supplied environment variables (#3 above) with `ARGOCD_ENV_`. This prevents users from directly setting 
    potentially-sensitive environment variables.

4. Parameters in the Application spec:

        apiVersion: argoproj.io/v1alpha1
        kind: Application
        spec:
         source:
           plugin:
             parameters:
               - name: values-files
                 array: [values-dev.yaml]
               - name: helm-parameters
                 map:
                   image.tag: v1.2.3
   
    The parameters are available as JSON in the `ARGOCD_APP_PARAMETERS` environment variable. The example above would
    produce this JSON:
   
        [{"name": "values-files", "array": ["values-dev.yaml"]}, {"name": "helm-parameters", "map": {"image.tag": "v1.2.3"}}]
   
    !!! note
        Parameter announcements, even if they specify defaults, are _not_ sent to the plugin in `ARGOCD_APP_PARAMETERS`.
        Only parameters explicitly set in the Application spec are sent to the plugin. It is up to the plugin to apply
        the same defaults as the ones announced to the UI.
   
    The same parameters are also available as individual environment variables. The names of the environment variables
    follows this convention:
   
           - name: some-string-param
             string: some-string-value
           # PARAM_SOME_STRING_PARAM=some-string-value
           
           - name: some-array-param
             value: [item1, item2]
           # PARAM_SOME_ARRAY_PARAM_0=item1
           # PARAM_SOME_ARRAY_PARAM_1=item2
           
           - name: some-map-param
             map:
               image.tag: v1.2.3
           # PARAM_SOME_MAP_PARAM_IMAGE_TAG=v1.2.3
   
> [!WARNING]
> **Sanitize/escape user input**
>
> As part of Argo CD's manifest generation system, config management plugins are treated with a level of trust. Be
> sure to escape user input in your plugin to prevent malicious input from causing unwanted behavior.

## Using a config management plugin with an Application

You may leave the `name` field
empty in the `plugin` section for the plugin to be automatically matched with the Application based on its discovery rules. If you do mention the name make sure 
it is either `<metadata.name>-<spec.version>` if version is mentioned in the `ConfigManagementPlugin` spec or else just `<metadata.name>`. When name is explicitly 
specified only that particular plugin will be used if its discovery pattern/command matches the provided application repo.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
    plugin:
      env:
        - name: FOO
          value: bar
```

If you don't need to set any environment variables, you can set an empty plugin section.

```yaml
    plugin: {}
```

> [!IMPORTANT]
> If your CMP command runs too long, the command will be killed, and the UI will show an error. The CMP server
> respects the timeouts set by the `server.repo.server.timeout.seconds` and `controller.repo.server.timeout.seconds` 
> items in `argocd-cmd-params-cm`. Increase their values from the default of 60s.
>
> Each CMP command will also independently timeout on the `ARGOCD_EXEC_TIMEOUT` set for the CMP sidecar. The default
> is 90s. So if you increase the repo server timeout greater than 90s, be sure to set `ARGOCD_EXEC_TIMEOUT` on the
> sidecar.
    
> [!NOTE]
> Each Application can only have one config management plugin configured at a time. If you're converting an existing
> plugin configured through the `argocd-cm` ConfigMap to a sidecar, make sure to update the plugin name to either `<metadata.name>-<spec.version>` 
> if version was mentioned in the `ConfigManagementPlugin` spec or else just use `<metadata.name>`. You can also remove the name altogether 
> and let the automatic discovery to identify the plugin.

> [!NOTE]
> If a CMP renders blank manifests, and `prune` is set to `true`, Argo CD will automatically remove resources. CMP plugin authors should ensure errors are part of the exit code. Commonly something like `kustomize build . | cat` won't pass errors because of the pipe. Consider setting `set -o pipefail` so anything piped will pass errors on failure.

> [!NOTE]
> If a CMP command fails to gracefully exit on `ARGOCD_EXEC_TIMEOUT`, it will be forcefully killed after an additional timeout of `ARGOCD_EXEC_FATAL_TIMEOUT`.

## Debugging a CMP

If you are actively developing a sidecar-installed CMP, keep a few things in mind:

1. If you are mounting plugin.yaml from a ConfigMap, you will have to restart the repo-server Pod so the plugin will
   pick up the changes.
2. If you have baked plugin.yaml into your image, you will have to build, push, and force a re-pull of that image on the
   repo-server Pod so the plugin will pick up the changes. If you are using `:latest`, the Pod will always pull the new
   image. If you're using a different, static tag, set `imagePullPolicy: Always` on the CMP's sidecar container.
3. CMP errors are cached by the repo-server in Redis. Restarting the repo-server Pod will not clear the cache. Always
   do a "Hard Refresh" when actively developing a CMP so you have the latest output.
4. Verify your sidecar has started properly by viewing the Pod and seeing that two containers are running `kubectl get pod -l app.kubernetes.io/component=repo-server -n argocd`
5. Write log message to stderr and set the `--loglevel=info` flag in the sidecar. This will print everything written to stderr, even on successful command execution.


### Other Common Errors
| Error Message | Cause |
| -- | -- |
| `no matches for kind "ConfigManagementPlugin" in version "argoproj.io/v1alpha1"` | The `ConfigManagementPlugin` CRD was deprecated in Argo CD 2.4 and removed in 2.8. This error means you've tried to put the configuration for your plugin directly into Kubernetes as a CRD. Refer to this [section of documentation](#write-the-plugin-configuration-file) for how to write the plugin configuration file and place it properly in the sidecar. |

## Plugin tar stream exclusions

In order to increase the speed of manifest generation, certain files and folders can be excluded from being sent to your
plugin. We recommend excluding your `.git` folder if it isn't necessary. Use Go's
[filepatch.Match](https://pkg.go.dev/path/filepath#Match) syntax. For example, `.git/*` to exclude `.git` folder.

You can set it one of three ways:

1. The `--plugin-tar-exclude` argument on the repo server.
2. The `reposerver.plugin.tar.exclusions` key if you are using `argocd-cmd-params-cm`
3. Directly setting `ARGOCD_REPO_SERVER_PLUGIN_TAR_EXCLUSIONS` environment variable on the repo server.

For option 1, the flag can be repeated multiple times. For option 2 and 3, you can specify multiple globs by separating
them with semicolons.

## Application manifests generation using argocd.argoproj.io/manifest-generate-paths

To enhance the application manifests generation process, you can enable the use of the `argocd.argoproj.io/manifest-generate-paths` annotation. When this flag is enabled, the resources specified by this annotation will be passed to the CMP server for generating application manifests, rather than sending the entire repository. This can be particularly useful for monorepos.

You can set it one of three ways:

1. The `--plugin-use-manifest-generate-paths` argument on the repo server.
2. The `reposerver.plugin.use.manifest.generate.paths` key if you are using `argocd-cmd-params-cm`
3. Directly setting `ARGOCD_REPO_SERVER_PLUGIN_USE_MANIFEST_GENERATE_PATHS` environment variable on the repo server to `true`.

## Migrating from argocd-cm plugins

Installing plugins by modifying the argocd-cm ConfigMap is deprecated as of v2.4 and has been completely removed starting in v2.8.

CMP plugins work by adding a sidecar to `argocd-repo-server` along with a configuration in that sidecar located at `/home/argocd/cmp-server/config/plugin.yaml`. A argocd-cm plugin can be easily converted with the following steps.

### Convert the ConfigMap entry into a config file

First, copy the plugin's configuration into its own YAML file. Take for example the following ConfigMap entry:

```yaml
data:
  configManagementPlugins: |
    - name: pluginName
      init:                          # Optional command to initialize application source directory
        command: ["sample command"]
        args: ["sample args"]
      generate:                      # Command to generate Kubernetes Objects in either YAML or JSON
        command: ["sample command"]
        args: ["sample args"]
      lockRepo: true                 # Defaults to false. See below.
```

The `pluginName` item would be converted to a config file like this:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: pluginName
spec:
  init:                          # Optional command to initialize application source directory
    command: ["sample command"]
    args: ["sample args"]
  generate:                      # Command to generate Kubernetes Objects in either YAML or JSON
    command: ["sample command"]
    args: ["sample args"]
```

> [!NOTE]
> The `lockRepo` key is not relevant for sidecar plugins, because sidecar plugins do not share a single source repo
> directory when generating manifests.

Next, we need to decide how this yaml is going to be added to the sidecar. We can either bake the yaml directly into the image, or we can mount it from a ConfigMap. 

If using a ConfigMap, our example would look like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pluginName
  namespace: argocd
data:
  pluginName.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: pluginName
    spec:
      init:                          # Optional command to initialize application source directory
        command: ["sample command"]
        args: ["sample args"]
      generate:                      # Command to generate Kubernetes Objects in either YAML or JSON
        command: ["sample command"]
        args: ["sample args"]
```

Then this would be mounted in our plugin sidecar.

### Write discovery rules for your plugin

Sidecar plugins can use either discovery rules or a plugin name to match Applications to plugins. If the discovery rule is omitted 
then you have to explicitly specify the plugin by name in the app spec or else that particular plugin will not match any app.

If you want to use discovery instead of the plugin name to match applications to your plugin, write rules applicable to 
your plugin [using the instructions above](#write-discovery-rules-for-your-plugin) and add them to your configuration 
file.

To use the name instead of discovery, update the name in your application manifest to `<metadata.name>-<spec.version>` 
if version was mentioned in the `ConfigManagementPlugin` spec or else just use `<metadata.name>`. For example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
spec:
  source:
    plugin:
      name: pluginName  # Delete this for auto-discovery (and set `plugin: {}` if `name` was the only value) or use proper sidecar plugin name
```

### Make sure the plugin has access to the tools it needs

Plugins configured with argocd-cm ran on the Argo CD image. This gave it access to all the tools installed on that
image by default (see the [Dockerfile](https://github.com/argoproj/argo-cd/blob/master/Dockerfile) for base image and
installed tools).

You can either use a stock image (like ubuntu, busybox, or alpine/k8s) or design your own base image with the tools your plugin needs. For
security, avoid using images with more binaries installed than what your plugin actually needs.

### Test the plugin

After installing the plugin as a sidecar [according to the directions above](#installing-a-config-management-plugin),
test it out on a few Applications before migrating all of them to the sidecar plugin.

Once tests have checked out, remove the plugin entry from your argocd-cm ConfigMap.

### Additional Settings

#### Preserve repository files mode

By default, config management plugin receives source repository files with reset file mode. This is done for security
reasons. If you want to preserve original file mode, you can set `preserveFileMode` to `true` in the plugin spec:

> [!WARNING]
> Make sure you trust the plugin you are using. If you set `preserveFileMode` to `true` then the plugin might receive
> files with executable permissions which can be a security risk.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: pluginName
spec:
  init:
    command: ["sample command"]
    args: ["sample args"]
  generate:
    command: ["sample command"]
    args: ["sample args"]
  preserveFileMode: true
```

##### Provide Git Credentials

By default, the config management plugin is responsible for providing its own credentials to additional Git repositories
that may need to be accessed during manifest generation. The reposerver has these credentials available in its git creds
store. When credential sharing is allowed, the git credentials used by the reposerver to clone the repository contents
are shared for the lifetime of the execution of the config management plugin, utilizing git's `ASKPASS` method to make a
call from the config management sidecar container to the reposerver to retrieve the initialized git credentials.

Utilizing `ASKPASS` means that credentials are not proactively shared, but rather only provided when an operation requires
them.

`ASKPASS` requires a socket to be shared between the config management plugin and the reposerver. To mitigate path traversal
attacks, it's recommended to use a dedicated volume to share the socket, and mount it in the reposerver and sidecar.
To change the socket path, you must set the `ARGOCD_ASK_PASS_SOCK` environment variable for both containers.

To allow the plugin to access the reposerver git credentials, you can set `provideGitCreds` to `true` in the plugin spec:

> [!WARNING]
> Make sure you trust the plugin you are using. If you set `provideGitCreds` to `true` then the plugin will receive
> credentials used to clone the source Git repository.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: pluginName
spec:
  init:
    command: ["sample command"]
    args: ["sample args"]
  generate:
    command: ["sample command"]
    args: ["sample args"]
  provideGitCreds: true
```




# ⛵ Cluster Template

Welcome to my opinionated and extensible template for deploying a single Kubernetes cluster. The goal of this project is to make it easier for people interested in using Kubernetes to deploy a cluster at home on bare-metal or VMs.

At a high level this project makes use of [makejinja](https://github.com/mirkolenz/makejinja) to read in a [configuration file](./config.sample.yaml) which renders out templates that will allow you to install and manage your Kubernetes cluster with.

## ✨ Features

The features included will depend on the type of configuration you want to use. There are currently **2 different types** of **configurations** available with this template.

1. **"Flux cluster"** - a Kubernetes cluster deployed on-top of [Talos Linux](https://github.com/siderolabs/talos) with an opinionated implementation of [Flux](https://github.com/fluxcd/flux2) using [GitHub](https://github.com/) as the Git provider and [sops](https://github.com/getsops/sops) to manage secrets.

    - **Required:** Some knowledge of [Containers](https://opencontainers.org/), [YAML](https://yaml.org/), and [Git](https://git-scm.com/).
    - **Components:** [flux](https://github.com/fluxcd/flux2), [Cilium](https://github.com/cilium/cilium),[cert-manager](https://github.com/cert-manager/cert-manager), [spegel](https://github.com/spegel-org/spegel), [reloader](https://github.com/stakater/Reloader), and [openebs](https://github.com/openebs/openebs).

2. **"Flux cluster with Cloudflare"** - An addition to "**Flux cluster**" that provides DNS and SSL with [Cloudflare](https://www.cloudflare.com/). [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/) is also included to provide external access to certain applications deployed in your cluster.

    - **Required:** A Cloudflare account with a domain managed in your Cloudflare account.
    - **Components:** [ingress-nginx](https://github.com/kubernetes/ingress-nginx/), [external-dns](https://github.com/kubernetes-sigs/external-dns) and [cloudflared](https://github.com/cloudflare/cloudflared).

**Other features include:**

- A [Renovate](https://www.mend.io/renovate)-ready repository with pull request diffs provided by [flux-local](https://github.com/allenporter/flux-local)
- Integrated [GitHub Actions](https://github.com/features/actions) with helpful workflows.

## 💻 Machine Preparation

### System requirements

> [!NOTE]
> 1. The included behaviour of Talos is that all nodes are able to run workloads, **including** the controller nodes. **Worker nodes** are therefore **optional**.
> 2. Do you have 3 or more nodes? It is highly recommended to make 3 of them controller nodes for a highly available control plane.
> 3. Running the cluster on Proxmox VE? My thoughts and recommendations about that are documented [here](https://onedr0p.github.io/home-ops/notes/proxmox-considerations.html).

| Role    | Cores    | Memory        | System Disk               |
|---------|----------|---------------|---------------------------|
| Control | 4 _(6*)_ | 8GB _(24GB*)_ | 120GB _(500GB*)_ SSD/NVMe |
| Worker  | 4 _(6*)_ | 8GB _(24GB*)_ | 120GB _(500GB*)_ SSD/NVMe |
| _\* recommended_ |

1. Head over to <https://factory.talos.dev> and follow the instructions which will eventually lead you to download a Talos Linux iso file (or for SBCs the `.raw.xz`). Make sure to note the schematic ID you will need this later on.

2. Flash the iso or raw file to a USB drive and boot to Talos on your nodes with it.

3. Continue on to 🚀 [**Getting Started**](#-getting-started)

## 🚀 Getting Started

Once you have installed Talos on your nodes, there are six stages to getting a Flux-managed cluster up and runnning.

> [!NOTE]
> For all stages below the commands **MUST** be ran on your personal workstation within your repository directory

### 🎉 Stage 1: Create a Git repository

1. Create a new **public** repository by clicking the big green "Use this template" button at the top of this page.

2. Clone **your new repo** to you local workstation and `cd` into it.

3. Continue on to 🌱 [**Stage 2**](#-stage-2-setup-your-local-workstation-environment)

### 🌱 Stage 2: Setup your local workstation

You have two different options for setting up your local workstation.

- First option is using a `devcontainer` which requires you to have Docker and VSCode installed. This method is the fastest to get going because all the required CLI tools are provided for you in my [devcontainer](https://github.com/onedr0p/cluster-template/pkgs/container/cluster-template%2Fdevcontainer) image.
- The second option is setting up the CLI tools directly on your workstation.

#### Devcontainer method

1. Start Docker and open your repository in VSCode. There will be a pop-up asking you to use the `devcontainer`, click the button to start using it.

2. Continue on to 🔧 [**Stage 3**](#-stage-3-bootstrap-configuration)

#### Non-devcontainer method

1. Install the most recent version of [task](https://taskfile.dev/), see the [installation docs](https://taskfile.dev/installation/) for other supported platforms.

    ```sh
    # Homebrew
    brew install go-task
    # or, Arch
    pacman -S --noconfirm go-task && ln -sf /usr/bin/go-task /usr/local/bin/task
    ```

2. Install the most recent version of [direnv](https://direnv.net/), see the [installation docs](https://direnv.net/docs/installation.html) for other supported platforms.

    ```sh
    # Homebrew
    brew install direnv
    # or, Arch
    pacman -S --noconfirm direnv
    ```

3. [Hook `direnv` into your preferred shell](https://direnv.net/docs/hook.html), then run:

    ```sh
    task workstation:direnv
    ```

    📍 _**Verify** that `direnv` is setup properly by opening a new terminal and `cd`ing into your repository. You should see something like:_

    ```sh
    cd /path/to/repo
    direnv: loading /path/to/repo/.envrc
    direnv: export +ANSIBLE_COLLECTIONS_PATH ...  +VIRTUAL_ENV ~PATH
    ```

4. Install the additional **required** CLI tools

   📍 _**Not using Homebrew or ArchLinux?** Try using the generic Linux task below, if that fails check out the [Brewfile](.taskfiles/Workstation/Brewfile)/[Archfile](.taskfiles/Workstation/Archfile) for what CLI tools needed and install them._

    ```sh
    # Homebrew
    task workstation:brew
    # or, Arch with yay/paru
    task workstation:arch
    # or, Generic Linux (YMMV, this pulls binaires in to ./bin)
    task workstation:generic-linux
    ```

5. Setup a Python virual environment by running the following task command.

    📍 _This commands requires Python 3.11+ to be installed._

    ```sh
    task workstation:venv
    ```

6. Continue on to 🔧 [**Stage 3**](#-stage-3-bootstrap-configuration)

### 🔧 Stage 3: Bootstrap configuration

> [!NOTE]
> The [config.sample.yaml](./config.sample.yaml) file contains config that is **vital** to the bootstrap process.

1. Generate the `config.yaml` from the [config.sample.yaml](./config.sample.yaml) configuration file.

    ```sh
    task init
    ```

2. Fill out the `config.yaml` configuration file using the comments in that file as a guide.

3. Run the following command which will generate all the files needed to continue.

    ```sh
    task configure
    ```

4. Push you changes to git

   📍 _**Verify** all the `./kubernetes/**/*.sops.*` files are **encrypted** with SOPS_

    ```sh
    git add -A
    git commit -m "Initial commit :rocket:"
    git push
    ```

### ⛵ Stage 4: Install Kubernetes

1. Deploy your cluster and bootstrap it. This generates secrets, generates the config files for your nodes and applies them. It bootstraps the cluster afterwards, fetches the kubeconfig file and installs Cilium and kubelet-csr-approver. It finishes with some health checks.

    ```sh
    task talos:bootstrap
    ```

2. ⚠️ It might take a while for the cluster to be setup (10+ minutes is normal), during which time you will see a variety of error messages like: "couldn't get current server API group list," "error: no matching resources found", etc. This is a normal. If this step gets interrupted, e.g. by pressing <kbd>Ctrl</kbd> + <kbd>C</kbd>, you likely will need to [nuke the cluster](#-Nuke) before trying again.

#### Cluster validation

1. The `kubeconfig` for interacting with your cluster should have been created in the root of your repository.

2. Verify the nodes are online

    📍 _If this command **fails** you likely haven't configured `direnv` as [mentioned previously](#non-devcontainer-method) in the guide._

    ```sh
    kubectl get nodes -o wide
    # NAME           STATUS   ROLES                       AGE     VERSION
    # k8s-0          Ready    control-plane,etcd,master   1h      v1.30.1
    # k8s-1          Ready    worker                      1h      v1.30.1
    ```

3. Continue on to 🔹 [**Stage 6**](#-stage-6-install-flux-in-your-cluster)

### 🔹 Stage 6: Install Flux in your cluster

1. Verify Flux can be installed

    ```sh
    flux check --pre
    # ► checking prerequisites
    # ✔ kubectl 1.30.1 >=1.18.0-0
    # ✔ Kubernetes 1.30.1 >=1.16.0-0
    # ✔ prerequisites checks passed
    ```

2. Install Flux and sync the cluster to the Git repository

    📍 _Run `task flux:github-deploy-key` first if using a private repository._

    ```sh
    task flux:bootstrap
    # namespace/flux-system configured
    # customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
    # ...
    ```

1. Verify Flux components are running in the cluster

    ```sh
    kubectl -n flux-system get pods -o wide
    # NAME                                       READY   STATUS    RESTARTS   AGE
    # helm-controller-5bbd94c75-89sb4            1/1     Running   0          1h
    # kustomize-controller-7b67b6b77d-nqc67      1/1     Running   0          1h
    # notification-controller-7c46575844-k4bvr   1/1     Running   0          1h
    # source-controller-7d6875bcb4-zqw9f         1/1     Running   0          1h
    ```

### 🎤 Verification Steps

_Mic check, 1, 2_ - In a few moments applications should be lighting up like Christmas in July 🎄

1. Output all the common resources in your cluster.

    📍 _Feel free to use the provided [kubernetes tasks](.taskfiles/Kubernetes/Taskfile.yaml) for validation of cluster resources or continue to get familiar with the `kubectl` and `flux` CLI tools._

    ```sh
    task kubernetes:resources
    ```

2. ⚠️ It might take `cert-manager` awhile to generate certificates, this is normal so be patient.

3. 🏆 **Congratulations** if all goes smooth you will have a Kubernetes cluster managed by Flux and your Git repository is driving the state of your cluster.

4. 🧠 Now it's time to pause and go get some motel motor oil ☕ and admire you made it this far!

## 📣 Flux w/ Cloudflare post installation

#### 🌐 Public DNS

The `external-dns` application created in the `networking` namespace will handle creating public DNS records. By default, `echo-server` and the `flux-webhook` are the only subdomains reachable from the public internet. In order to make additional applications public you must set set the correct ingress class name and ingress annotations like in the HelmRelease for `echo-server`.

#### 🏠 Home DNS

`k8s_gateway` will provide DNS resolution to external Kubernetes resources (i.e. points of entry to the cluster) from any device that uses your home DNS server. For this to work, your home DNS server must be configured to forward DNS queries for `${bootstrap_cloudflare.domain}` to `${bootstrap_cloudflare.gateway_vip}` instead of the upstream DNS server(s) it normally uses. This is a form of **split DNS** (aka split-horizon DNS / conditional forwarding).

> [!TIP]
> Below is how to configure a Pi-hole for split DNS. Other platforms should be similar.
> 1. Apply this file on the Pihole server while substituting the variables
> ```sh
> # /etc/dnsmasq.d/99-k8s-gateway-forward.conf
> server=/${bootstrap_cloudflare.domain}/${bootstrap_cloudflare.gateway_vip}
> ```
> 2. Restart dnsmasq on the server.
> 3. Query an internal-only subdomain from your workstation (any `internal` class ingresses): `dig @${home-dns-server-ip} echo-server-internal.${bootstrap_cloudflare.domain}`. It should resolve to `${bootstrap_cloudflare.ingress_vip}`.

If you're having trouble with DNS be sure to check out these two GitHub discussions: [Internal DNS](https://github.com/onedr0p/cluster-template/discussions/719) and [Pod DNS resolution broken](https://github.com/onedr0p/cluster-template/discussions/635).

... Nothing working? That is expected, this is DNS after all!

#### 📜 Certificates

By default this template will deploy a wildcard certificate using the Let's Encrypt **staging environment**, which prevents you from getting rate-limited by the Let's Encrypt production servers if your cluster doesn't deploy properly (for example due to a misconfiguration). Once you are sure you will keep the cluster up for more than a few hours be sure to switch to the production servers as outlined in `config.yaml`.

📍 _You will need a production certificate to reach internet-exposed applications through `cloudflared`._

#### 🪝 Github Webhook

By default Flux will periodically check your git repository for changes. In order to have Flux reconcile on `git push` you must configure Github to send `push` events to Flux.

> [!NOTE]
> This will only work after you have switched over certificates to the Let's Encrypt Production servers.

1. Obtain the webhook path

    📍 _Hook id and path should look like `/hook/12ebd1e363c641dc3c2e430ecf3cee2b3c7a5ac9e1234506f6f5f3ce1230e123`_

    ```sh
    kubectl -n flux-system get receiver github-receiver -o jsonpath='{.status.webhookPath}'
    ```

2. Piece together the full URL with the webhook path appended

    ```text
    https://flux-webhook.${bootstrap_cloudflare.domain}/hook/12ebd1e363c641dc3c2e430ecf3cee2b3c7a5ac9e1234506f6f5f3ce1230e123
    ```

3. Navigate to the settings of your repository on Github, under "Settings/Webhooks" press the "Add webhook" button. Fill in the webhook URL and your `bootstrap_github_webhook_token` secret in `config.yaml`, Content type: `application/json`, Events: Choose Just the push event, and save.

## 💥 Nuke

There might be a situation where you want to destroy your Kubernetes cluster. The following command will reset your nodes back to maintenance mode, append `--force` to completely format your the Talos installation. Either way the nodes should reboot after the command has run.

```sh
task talos:nuke
```

## 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions, Ansible roles... even Flux itself! Merging a PR will cause Flux to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.github/renovate.json5](./.github/renovate.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs right away.

## 🐛 Debugging

Below is a general guide on trying to debug an issue with an resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state.

1. Start by checking all Flux Kustomizations & Git Repository & OCI Repository and verify they are healthy.

    ```sh
    flux get sources oci -A
    flux get sources git -A
    flux get ks -A
    ```

2. Then check all the Flux Helm Releases and verify they are healthy.

    ```sh
    flux get hr -A
    ```

3. Then check the if the pod is present.

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

4. Then check the logs of the pod if its there.

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    # or
    stern -n <namespace> <fuzzy-name>
    ```

5. If a resource exists try to describe it to see what problems it might have.

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

6. Check the namespace events

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be a external factor like permissions on NFS. If you are unable to figure out your problem see the help section below.

## ⬆️ Upgrading Talos and Kubernetes

### Manual

```sh
# Upgrade Talos to a newer version
# NOTE: This needs to be run once on every node
task talos:upgrade node=? image=?
# e.g.
# task talos:upgrade node=192.168.42.10 image=factory.talos.dev/installer/${schematic_id}:v1.7.4
```

```sh
# Upgrade Kubernetes to a newer version
# NOTE: This only needs to be run once against a controller node
task talos:upgrade-k8s controller=? to=?
# e.g.
# task talos:upgrade-k8s controller=192.168.42.10 to=1.30.1
```

## 👉 Help

- Make a post in this repository's Github [Discussions](https://github.com/onedr0p/cluster-template/discussions).
- Start a thread in the `#support` or `#cluster-template` channels in the [Home Operations](https://discord.gg/home-operations) Discord server.

## ❔ What's next

The cluster is your oyster (or something like that). Below are some optional considerations you might want to review.

### Ship it

To browse or get ideas on applications people are running, community member [@whazor](https://github.com/whazor) created [Kubesearch](https://kubesearch.dev) as a creative way to search Flux HelmReleases across Github and Gitlab.

### Storage

The included CSI (openebs in local-hostpath mode) is a great start for storage but soon you might find you need more features like replicated block storage, or to connect to a NFS/SMB/iSCSI server. If you need any of those features be sure to check out the projects like [rook-ceph](https://github.com/rook/rook), [longhorn](https://github.com/longhorn/longhorn), [openebs](https://github.com/openebs/openebs), [democratic-csi](https://github.com/democratic-csi/democratic-csi), [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs),
and [synology-csi](https://github.com/SynologyOpenSource/synology-csi).

## 🙌 Related Projects

If this repo is too hot to handle or too cold to hold check out these following projects.

- [khuedoan/homelab](https://github.com/khuedoan/homelab) - _Modern self-hosting framework, fully automated from empty disk to operating services with a single command._
- [danmanners/aws-argo-cluster-template](https://github.com/danmanners/aws-argo-cluster-template) - _A community opinionated template for deploying Kubernetes clusters on-prem and in AWS using Pulumi, SOPS, Sealed Secrets, GitHub Actions, Renovate, Cilium and more!_
- [ricsanfre/pi-cluster](https://github.com/ricsanfre/pi-cluster) - _Pi Kubernetes Cluster. Homelab kubernetes cluster automated with Ansible and ArgoCD_
- [techno-tim/k3s-ansible](https://github.com/techno-tim/k3s-ansible) - _The easiest way to bootstrap a self-hosted High Availability Kubernetes cluster. A fully automated HA k3s etcd install with kube-vip, MetalLB, and more_

## ⭐ Stargazers

<div align="center">

[![Star History Chart](https://api.star-history.com/svg?repos=onedr0p/cluster-template&type=Date)](https://star-history.com/#onedr0p/cluster-template&Date)

</div>

## 🤝 Thanks

Big shout out to all the contributors, sponsors and everyone else who has helped on this project.