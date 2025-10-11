# toolhive-webhook

A Kubernetes admission webhook for [ToolHive](https://github.com/stacklok/toolhive) MCPServer resources that automatically injects Kagenti client registration initContainer to enable registration of the server with Keycloak.

## Description

This webhook provides automatic client registration for MCPServer resources deployed in Kubernetes. When enabled, it mutates MCPServer Custom Resource (CR) to include an initContainer that registers the server as a client in Keycloak before the main container starts. This enables secure, automated service authentication within the Kagenti platform.

The webhook supports:

- Automatic injection of `kagenti-client-registration` initContainer

- Configurable client registration via the `--enable-client-registration` flag

- Shared volume mounting for credential propagation

- Integration with cert-manager for webhook TLS certificates

### Configuration

The webhook supports the following configuration options:

- `--enable-client-registration`: Enable automatic client registration in Keycloak (default: true)
- `--webhook-cert-path`: Directory containing webhook TLS certificates (default: auto-generated)
- `--webhook-cert-name`: Webhook certificate filename (default: tls.crt)
- `--webhook-cert-key`: Webhook key filename (default: tls.key)

The webhook requires a ConfigMap named `environments` in the same namespace with the following keys:

- `KEYCLOAK_URL`: Keycloak server URL
- `KEYCLOAK_REALM`: Keycloak realm name
- `KEYCLOAK_ADMIN_USERNAME`: Admin username for client registration
- `KEYCLOAK_ADMIN_PASSWORD`: Admin password for client registration


## Getting Started

### Prerequisites

- go version v1.24.4+
- docker version 17.03+.
- kubectl version v1.11.3+.
- Access to a Kubernetes v1.11.3+ cluster.

### To Deploy on the cluster

**Build and push your image to the location specified by `IMG`:**

```sh
make docker-build docker-push IMG=<some-registry>/toolhive-webhook:tag
```

**NOTE:** This image ought to be published in the personal registry you specified.
And it is required to have access to pull the image from the working environment.
Make sure you have the proper permission to the registry if the above commands don’t work.

**Install the CRDs into the cluster:**

```sh
make install
```

**Deploy the Manager to the cluster with the image specified by `IMG`:**

```sh
make deploy IMG=<some-registry>/toolhive-webhook:tag
```

> **NOTE**: If you encounter RBAC errors, you may need to grant yourself cluster-admin
privileges or be logged in as admin.

**Create instances of your solution**
You can apply the samples (examples) from the config/sample:

```sh
kubectl apply -k config/samples/
```

>**NOTE**: Ensure that the samples has default values to test it out.

### To Uninstall

**Delete the instances (CRs) from the cluster:**

```sh
kubectl delete -k config/samples/
```

**Delete the APIs(CRDs) from the cluster:**

```sh
make uninstall
```

**UnDeploy the controller from the cluster:**

```sh
make undeploy
```

## Project Distribution

Following the options to release and provide this solution to the users.

### By providing a bundle with all YAML files

1. Build the installer for the image built and published in the registry:

```sh
make build-installer IMG=<some-registry>/toolhive-webhook:tag
```

**NOTE:** The makefile target mentioned above generates an 'install.yaml'
file in the dist directory. This file contains all the resources built
with Kustomize, which are necessary to install this project without its
dependencies.

2. Using the installer

Users can just run 'kubectl apply -f <URL for YAML BUNDLE>' to install
the project, i.e.:

```sh
kubectl apply -f https://raw.githubusercontent.com/<org>/toolhive-webhook/<tag or branch>/dist/install.yaml
```

### By providing a Helm Chart

1. Build the chart using the optional helm plugin

```sh
kubebuilder edit --plugins=helm/v1-alpha
```

2. See that a chart was generated under 'dist/chart', and users
can obtain this solution from there.

**NOTE:** If you change the project, you need to update the Helm Chart
using the same command above to sync the latest changes. Furthermore,
if you create webhooks, you need to use the above command with
the '--force' flag and manually ensure that any custom configuration
previously added to 'dist/chart/values.yaml' or 'dist/chart/manager/manager.yaml'
is manually re-applied afterwards.

## Contributing

// TODO(user): Add detailed information on how you would like others to contribute to this project

**NOTE:** Run `make help` for more information on all potential `make` targets

More information can be found via the [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## License

Copyright 2025.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

