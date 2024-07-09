# Getting started

- [Getting started](#getting-started)
  - [Configuring ArgoCD Applications](#configuring-argocd-applications)
  - [Accessing ArgoCD](#accessing-argocd)
  - [Helpful tips](#helpful-tips)
    - [Debugging with the Helm template command](#debugging-with-the-helm-template-command)
    - [Common Issues](#common-issues)
  - [How does it work?](#how-does-it-work)

## Configuring ArgoCD Applications
There are two primary directories that will be used to interface with LHDI's instance of ArgoCD:

- An `apps` directory containing each Application's `config.json`

  ```text
  apps/
      httpbin/ <-- each Application requires a subdirectory under apps/
          config.json <-- Application configuration (app.name and app.path are required)
  ```
    - The [config.json](https://laughing-adventure-wllepqe.pages.github.io/Lighthouse-delivery-infrastructure-tools/Using-the-DI-tools/Step-4-Deploy-your-project-to-dev/Deploy-with-Argo-CD/#current-limitations) file **must** define the `name` of the Application and the `path` to a Helm directory containing the Application's deployment manifest.
      ```json
      {
          "app": {
            "name": "httpbin",
            "path": "deploy/httpbin"
          }
      }
      ```

      We **strongly** suggest keeping Application names short or abbreviated, if possible. Application names will be concatenated with other variables like team name and environment to avoid naming collisions, however, Application names must be **53 characters or less**. Longer Application names can easily break this threshold causing unexpected results.

- A `deploy` directory containing each Application's deployment manifest files.

  ```text
  deploy/
      httpbin/ <-- subdirectory for each Appication (this should match the "name" and "path" defined in the config.json)
          templates/ <-- Additional Kubernetes manifest files
              secrets.yaml 
              virtualService.yaml 
          Chart.yaml
          dev.yaml  <-- environment specific overrides for the Dev environment
          qa.yaml <-- environment specific overrides for the QA environment
          sandbox.yaml <-- environment specific overrides for the Sandbox environment
          prod.yaml <-- environment specific overrides for the Prod environment
          values.yaml <-- global configuation values
  ```

    > For more information about using Helm Charts see [Helm Docs](https://helm.sh/docs/topics/charts/)

## Accessing ArgoCD

For more information about using the ArgoCD UI or CLI, visit the [Accessing ArgoCD](https://laughing-adventure-wllepqe.pages.github.io/Lighthouse-delivery-infrastructure-tools/Using-the-DI-tools/Step-4-Deploy-your-project-to-dev/Deploy-with-Argo-CD/#accessing-argo-cd) section on [Platform Delivery Docs](https://laughing-adventure-wllepqe.pages.github.io)

## Helpful tips

### Debugging with the Helm template command

**Overview:** 
ArgoCD uses the `helm template` command to generate deployment manifests. The same command can be used locally to validate the syntax of any Application's deployment manifests for any environment before the Application is deployed. 

**Why is it helpful?:** 
When ArgoCD fails to generate a manifest, the ArgoCD UI may show a healthy Application without any resources. Using the `helm template` command can be a helpful first debugging step if you encounter an Application that fails to generate the expected resources.

First build any external dependencies: 

```sh
$ helm dependency build deploy/httpbin/
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "estahn" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading httpbingo from repo https://estahn.github.io/charts/
Deleting outdated charts
```

Now the deployment manifest can be templated using the command below:
```sh
$ helm template lint-check deploy/httpbin/ \
    --debug \
    -f deploy/httpbin/values.yaml \
    -f deploy/httpbin/dev.yaml 
```

In some cases, a [mutating webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks) may be the cause for ArgoCD failing to generate resources with the Application manifest. In this case, you can use `kubectl apply --dry-run=client` to invoke any mutating functions that are being applied to the Application manifest.

```sh
$ helm template lint-check deploy/httpbin/ \
    --debug \
    -f deploy/httpbin/values.yaml \
    -f deploy/httpbin/dev.yaml \
    | kubectl apply --dry-run=client -n <team-name>-dev -f - 
...
error: error validating "STDIN": error validating data: unknown object type
"nil" in DBInstance.metadata.annotations.iaas.lighthouse.va.gov/service-account;
if you choose to ignore these errors, turn validation off with --validate=false
```
**Additional information:**
Values files for Helm commands take precedence based on order. 

In the commands shown above, the `values.yaml` is applied first, then a `dev.yaml` file containing environment specific values for the `dev` environment is applied. Any values defined in the `dev.yaml` will override the corresponding values defined in the `values.yaml`. 

### Common Issues

See other troubleshooting steps on our [Common Issues](https://laughing-adventure-wllepqe.pages.github.io/Lighthouse-delivery-infrastructure-tools/Using-the-DI-tools/Step-4-Deploy-your-project-to-dev/Deploy-with-Argo-CD/#common-issues) section on [Platform Delivery Docs](https://laughing-adventure-wllepqe.pages.github.io/).

## How does it work?

> Note: the following is additional information about how this repository works behind the scenes with ArgoCD.

When a team is onboarded to the LHDI Platform, an ArgoCD `Project` and `ApplicationSet` are created for the team. The `Project` defines permissions for an `ApplicationSet`, and an `ApplicationSet` can generate applications for one or more environments.

The `ApplicationSet` created by LHDI will look similar to the following:

```yaml
# example application set
spec:
  generators:
  - matrix:
      generators:
      - list:
          elements:
          - environment: dev
          - environment: qa
          - environment: sandbox
          - environment: prod
      - git:
          files:
          - path: apps/*/config.json
          repoURL: https://github.com/department-of-veterans-affairs/<team-name>-argocd-applications-vault
          revision: HEAD
```

These generators instruct ArgoCD where it should look to find the applications and which environments to deploy these applications to. In this case, ArgoCD will look at the Git repository `https://github.com/department-of-veterans-affairs/<team-name>-argocd-applications-vault` and identify each application using the file(s) that match the path `apps/*/config.json`.

Each of these `config.json` files will define the `name` and `path` for an application manifest. 

For each application manifest under `deploy/`, ArgoCD will deploy the application to each of the environments listed under `list.elements`. 
