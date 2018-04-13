# wordpress-helm-apb

This APB augments the stable helm chart for wordpress by:
- adding rich parameters for use in the Service Catalog
- adding an OpenShift Route when provisioning on OpenShift

## Cluster Permissions

Because the wordpress image used by this chart [requires running as the root
user](https://github.com/bitnami/bitnami-docker-wordpress/issues/126), for now
you must run the following command in the target namespace when using
OpenShift. It allows processes in your target namespace to run as any user,
including root, so keep that in mind. The wordpress container will run as root
unless the issue linked above has been resolved.

```
$ oc adm policy add-scc-to-user anyuid -z default
```

## Database

This APB utilizes a separately-provisioned MariaDB database. At provision time,
it expects to find a secret with label `app: wordpress` in the target
namespace. That secret must contain these keys:

* DB_HOST
* DB_USER
* DB_PASSWORD
* DB_NAME
* DB_PORT

One way to fulfill that database need is to:

1. Provision the mariadb-apb
2. Create a binding
3. Edit the binding secret to add the label `app: wordpress`

## Roles

The following sections are only valuable to APB developers who want to
understand how best to use custom roles with the helm-ansible-base image.

### Provision

Provisioning utilizes three roles, shown below.

`helm-ansible-base` is provided by the base image. It uses the helm chart to
create a manifest and create the corresponding resources in kubernetes.

The other two roles provide an opportunity to take action before and after the
helm-based provisioning.


```
  - role: preprovision-wordpress
  - role: helm-ansible-base
  - role: postprovision-wordpress
```

`preprovision-wordpress` finds the secret with a database binding and makes
its data available as facts.

`postprovision-wordpress` creates an OpenShift Route.

### Deprovision

Deprovisioning utilized only the `helm-ansible-base` role, which is provided by
the base image. It deletes all resources in the namespace that have the label
corresponding to the helm "release" identifier. An example label would be
`release: bundle-c918f162`. Helm automatically puts that label on each resource
in the manifest it generates.

The OpenShift route created by the `postprovision-wordpress` role also includes
the same label, so it too will be auto-deleted by the base role.
