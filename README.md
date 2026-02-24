# Red Hat Dev Spaces extended with AI

Red Hat OpenShift Dev Spaces workspace with the [Continue](https://continue.dev) AI assistant pre-installed.

The following AI extensions are included:

* Use of [Continue.dev](https://www.continue.dev/) extension

## What's included

- **devfile.yaml** – Dev Space definition with VS Code, adding extension.
- **.vscode/extensions.json** – Declare extensions to install (via `.vscode/extensions.json`).
- **.continue/config.yaml** – Continue configuration template.

## Environment variables in the workstation

When a workspace is created from this `devfile`, the following environment variables are available inside
the containers (e.g. in the terminal and to commands/scripts).

### Reserved variables (set by the devfile / Dev Workspace runtime)

These are defined by the [devfile specification](https://devfile.io/docs/2.2.0/defining-environment-variables) 
and **cannot be overridden** in the devfile `env` section.
They are set automatically in every container that has `mountSources: true`:

| Variable           | Description |
|--------------------|-------------|
| **`PROJECTS_ROOT`** | Path where project sources are mounted. Defined by the container’s `sourceMapping`; default is **`/projects`**. Use this in commands and scripts to refer to the root of cloned projects. |
| **`PROJECT_SOURCE`** | Path to the **first** project’s source directory. Equals `$PROJECTS_ROOT/<project-name>`. With a single project, this is typically `/projects/<repo-name>`. |

You can use `$PROJECTS_ROOT` and `$PROJECT_SOURCE` in devfile **commandLine** and **workingDir** (e.g. in `postStart` commands).

### Custom Variables defined

Continue is configured using environment variables for the **model API key** (`CONTINUE_API_KEY`)
and the **model API base URL** (`CONTINUE_API_URL`), both provided via an OpenShift Secret mounted
into the Dev Space workstation.

| Variable               | Description |
|------------------------|-------------|
| **`CONTINUE_API_KEY`** | Model API key for the Continue extension. Provided by mounting a Secret with `mount-as: env` (see below); the devfile declares it with an empty default. |
| **`CONTINUE_API_URL`** | Model API base URL (e.g. `https://llama-3-2-3b-maas-apicast-production.apps.prod.rhoai.rh-aiservices-bu.com:443/v1`). Provided by the same Secret; substituted into `apiBase` in the copied Continue config. |

### Variables from Secrets and ConfigMaps

Any Kubernetes Secret or ConfigMap that you mount into the Dev Workspace with **`controller.devfile.io/mount-as=env`**
adds its keys as environment variable names and its values as their values. This `devfile` expects a
Secret with keys **`CONTINUE_API_KEY`** and **`CONTINUE_API_URL`** (see below).

### Other environment variables

- **Standard container env**: The image and the platform set common variables such as `HOME` (e.g. `/home/user`), `USER`, `PATH`, etc.
- **Implementation-specific**: Red Hat Dev Spaces may inject additional variables (e.g. workspace or user identifiers). For an authoritative list for your version, see the [Dev Spaces documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_dev_spaces/) or the [devfile API schema](https://devfile.io/docs/2.2.0/devfile-schema).

To inspect variables at runtime, run `env` or `printenv` in a terminal inside the workspace.

## Setting the API key and API URL as environment variables

The devfile expects some environment variables, both provided by a single **Kubernetes Secret** mounted as env:

- **`CONTINUE_MODEL_NAME`** - Model name for Contiue.
- **`CONTINUE_MODEL_API_URL`** – Model API base URL (e.g. `https://llama-3-2-3b-maas-apicast-production.apps.prod.rhoai.rh-aiservices-bu.com:443/v1`). This is written into the copied Continue config as `apiBase`.
- **`CONTINUE_MODEL_API_KEY`** – Model API key for Continue.

Create the Secret in your **user namespace** and mount it with the labels and annotation
below so both variables are available in the workstation. The postStart command then
copies `.continue/config.yaml` to `~/.continue/config.yaml` and replaces the env variables.

### Prerequisites

- Access to the OpenShift cluster where Dev Spaces is running.
- Your Dev Spaces user namespace (e.g. `https://<devspaces-host>/api/kubernetes/namespace`).
- `oc` logged in with permissions to create resources in that namespace.

### Step 1: Create a Secret

Create a Secret in your **user namespace** with keys needed so they match the devfile and
the Continue config template.

**Option A – Using `oc` (recommended)**

```bash
# Replace <your-namespace> with your Dev Spaces user namespace
# Replace <your-api-key> with your actual API key
# Replace <your-api-url> with your model API base URL (e.g. https://.../v1)
oc create secret generic continue-api-secret \
  --from-literal=CONTINUE_MODEL_NAME='<your-model-name>' \
  --from-literal=CONTINUE_MODEL_API_URL='<your-api-url>' \
  --from-literal=CONTINUE_MODEL_API_KEY='<your-api-key>' \
  -n <your-namespace>
```

Example with a Red Hat OpenShift AI (RHOAI) endpoint:

```bash
oc create secret generic continue-api-secret \
  --from-literal=CONTINUE_MODEL_NAME='llama-3-2-3b' \
  --from-literal=CONTINUE_MODEL_API_URL='https://llama-3-2-3b-maas-apicast-production.apps.prod.rhoai.rh-aiservices-bu.com:443/v1' \
  --from-literal=CONTINUE_MODEL_API_KEY='<your-api-key>' \
  -n <your-namespace>
```

**Option B – Using a manifest**

1. Encode the key and URL in Base64 (no trailing newline):

   ```bash
   echo -n '<your-api-key>' | base64
   echo -n '<your-api-url>' | base64
   ```

2. Create a file (e.g. `continue-secret.yaml`) and apply it in your user namespace:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: continue-api-secret
     namespace: <your-namespace>
   type: Opaque
   data:
     CONTINUE_MODEL_NAME: <base64-encoded-model-name>
     CONTINUE_API_URL: <base64-encoded-api-url>
     CONTINUE_API_KEY: <base64-encoded-api-key>
   ```

   ```bash
   oc apply -f continue-secret.yaml
   ```

### Step 2: Mount the Secret as environment variables

Add the required labels and the `mount-as: env` annotation so the Secret is mounted
as environment variables in all Dev Workspace containers:

```bash
oc label secret continue-api-secret \
  controller.devfile.io/mount-to-devworkspace=true \
  controller.devfile.io/watch-secret=true \
  -n <your-namespace>

oc annotate secret continue-api-secret \
  controller.devfile.io/mount-as=env \
  -n <your-namespace>
```

- **mount-to-devworkspace=true** – Mounts the Secret into your Dev Workspace.
- **watch-secret=true** – Changes are picked up when you restart the workspace.
- **mount-as=env** – Each Secret key becomes an environment variable (key name = variable name).

### Step 3: Start or restart your workspace

1. Start a new workspace from this repository, or restart an existing one that uses this devfile.
2. Environment variables will be available in the workstation.
3. The postStart command copies `.continue/config.yaml` to `~/.continue/config.yaml` and substitutes both values; Continue uses that file.

### Verification

Inside the Dev Space, open a terminal and run:

```bash
echo "CONTINUE_MODEL_NAME is set: $(if [ -n \"$CONTINUE_MODEL_NAME\" ]; then echo yes; else echo no; fi)"
echo "CONTINUE_MODEL_API_URL is set: $(if [ -n \"$CONTINUE_MODEL_API_URL\" ]; then echo yes; else echo no; fi)"
echo "CONTINUE_MODEL_API_KEY is set: $(if [ -n \"$CONTINUE_MODEL_API_KEY\" ]; then echo yes; else echo no; fi)"
```

Continue plugin is installed and you can interact with the Model defined in your secret.
This is a sample screenshot with an open chat with the model.

![Continue Chat](./images/continue-chat.png)

## References

- [Continue – Configuration](https://docs.continue.dev/customize/deep-dives/configuration)
- [Continue – config.yaml reference](https://docs.continue.dev/reference/config)
- [Red Hat Dev Spaces – Using credentials and configurations in workspaces](https://docs.redhat.com/en/documentation/red_hat_openshift_dev_spaces/latest/html/user_guide/using-credentials-and-configurations-in-workspaces)
