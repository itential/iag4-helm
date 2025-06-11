# Helm charts for Itential Automation Gateway 4

This repo contains helm charts for running Itential Automation Platform and Itential Automation Gateway in Kubernetes. This repo contains two charts: iap and iag.

## Itential Automation Gateway (IAG)

The application is installed using a Kubernetes statefulset. It also includes persistent volume claims, ingress, and other Kubernetes objects suitable to run the application.

### Usage

```bash
helm install iag ./iag`
```

This will install IAG according to how its configured in the values.yaml file ("latest").

```bash
helm install iag ./iag --set image.tag=2023.2.7
```

This will install IAG with the "2023.2.7" image.

### Volumes

| Name | Type  | Description  |
|------|-------|--------------|
| config-volume   | Configmap               | Configuration properties for the IAG properties.yaml file. This is the main config file for the IAG application.         |
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

### How to add TLS/SSL certificates to the container

To add additional TLS or SSL certificates to the container created by Kubernetes, add any certificates to the `files/certificates` directory in either the IAP or IAG projects. Any certificates that are placed in that directory will be added to the container at `/etc/ssl/certs`.
