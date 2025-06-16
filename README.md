# Helm charts for Itential Automation Gateway 4

This repo contains helm charts for running Itential Automation Gateway in Kubernetes.

## Itential Automation Gateway (IAG)

The application is installed using a Kubernetes statefulset. It also includes persistent volume claims, ingress, and other Kubernetes objects suitable to run the application.

### Usage

```bash
helm install iag . -f values.yaml
```

This will install IAG according to how its configured in the values.yaml file.

```bash
helm install iag . -f values.yaml --set image.tag=4.3.4
```

This will install IAG with the "4.3.4" image.

### Requirements & Dependencies

#### Certificates

The chart will require a Certificate Authority to be added to the Kubernetes environment. This is
used by the chart when running with `useTLS` flag enabled. The chart will use this CA to generate
the necessary certificates using a Kubernetes `Issuer` which is included. The Issuer will issue the
certificates using the CA. The certificates are then included using a Kubernetes `Secret` which is
mounted by the pods. Creating and adding this CA is outside of the scope of this chart.

Both the `Issuer` and the `Certificate` objects are realized by using the widely used Kubernetes
add-on called `cert-manager`. Cert-manager is responsible for making the TLS certificates required
by using the CA that was installed separately. The installation of cert-manager is outside the scope
of this chart. To check if this is already installed run this command:

```bash
kubectl get crds | grep cert-manager
```

#### DNS

This is not a requirement but more of an explanation.

Itential used the ExternalDNS project to facilitate the creation of DNS entries. ExternalDNS
synchronizes exposed Kubernetes Services and Ingresses with DNS providers. This is not a
requirement for running the chart. The chart and the application can run without this if needed. We
chose to use this to provide us with external addresses.

For more information see the [ExternalDNS project](https://github.com/kubernetes-sigs/external-dns).

### Volumes

The application does require some volumes to exist. These are mounted and used by the application
as the location for custom scripts and for the location of the onboard database. They should persist
after a shutdown to prevent any data loss.

| Name | Type  | Description  |
|:-----|:------|:-------------|
| config-volume   | Configmap               | The chart uses this volume to mount important configurations files like ansible.cfg |
| iag-data-volume | Persistent Volume Claim | This represents the claim for the persistence for the sqlite data required by IAG.                                       |
| iag-code-volume | Persistent Volume Claim | This represents the claim for the location of all the customer authored scripts, ansible code, and other customizations. |

#### How to construct the iag-code-volume

This volume is intended to store any custom assets that the user is bringing to IAG. Its contents will reflect a customer's unique usage of IAG. There is an expectation in the container of the structure of the files in this volume. It should look like this:

```bash
.
├── ansible/ # custom ansible assets
│   ├── collections
│   ├── inventory
│   ├── modules
│   ├── playbooks
│   └── roles
├── venv/ # custom python virtual environments
├── nornir/
│   ├── conf
│   ├── inventory
│   └── modules
└── scripts/ # custom script assets
```

### Tests

The chart comes with a suite of unit tests. Those can be executed by running:

```bash
helm unittest . -f values.yaml
```
