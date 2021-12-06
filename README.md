# dev-team-alpha

This repo serves as an example of how to structure a tenant Git repository when leveraging [Hierarchical Namespaces].

The [Platform Admin] already [onboared the tenant] assigning it the namespace `dev-team-alpha-ns` and the service account `system:serviceaccount:dev-team-alpha-ns:flux`.

The hnc multi-tenant example configures this repo as a [gitrepository for a tenant here].

### Structure Caveats
Some structural decisions were intentional and are highly recommended:

- All flux objects are deployed at the top-level namespace, as it enables:
    a) Separation of duties: the tenant's top namespace is solely responsible for its flux's configuration.
    b) RBAC isolation from applications: the flux `ServiceAccount` used to apply changes is not propagated to subnamespaces. Therefore removing the risk of it being used by users with `pod create` permissions in subnamespaces.

- Tenants must always split the creation and use of `SubNamespace` objects in different `Kustomization` objects. This example creates `dev-team-alpha-subnamespaces` to create all its subnamespaces, and `dev-team-alpha-resources` to create everything else. This is required as kustomize is unaware that [Hierarchical Namespaces] will propagate the RBAC permissions to the subnamespaces automatically, therefore attempting to do both operations in a single `Kustomization` will fail during the `dry-run` due to "lack of permissions".


### Inspection of deployed repo

#### Namespaces

Once reconciled, three namespaces should be available:

```shell
$ kubectl get namespace dev-team-alpha-ns apps some-other-ns
NAME                STATUS   AGE
dev-team-alpha-ns   Active   56m
apps                Active   55m
some-other-ns       Active   55m
```

Using the [hnc plugin] they should structured as a tree:
```
$ kubectl hns tree dev-team-alpha-ns
dev-team-alpha-ns
├── [s] apps
└── [s] some-other-ns
```

#### Permissions

The `ServiceAccount` and `RoleBinding` defined by the Platform Admin is created at the top level:

```shell
$ kubectl get rolebinding,serviceaccount --namespace dev-team-alpha-ns
NAME                                                              ROLE                        AGE
rolebinding.rbac.authorization.k8s.io/dev-team-alpha-reconciler   ClusterRole/cluster-admin   34m

NAME                     SECRETS   AGE
serviceaccount/default   1         34m
serviceaccount/flux      1         34m
```

Only the `RoleBinding` is propagated downstream:
```shell
$ kubectl get rolebinding,serviceaccount --namespace apps
NAME                                                              ROLE                        AGE
rolebinding.rbac.authorization.k8s.io/dev-team-alpha-reconciler   ClusterRole/cluster-admin   34m

NAME                     SECRETS   AGE
serviceaccount/default   1         34m


$ kubectl get rolebinding,serviceaccount --namespace some-other-ns
NAME                                                              ROLE                        AGE
rolebinding.rbac.authorization.k8s.io/dev-team-alpha-reconciler   ClusterRole/cluster-admin   34m

NAME                     SECRETS   AGE
serviceaccount/default   1         34m
```

#### Applications

```shell
$ kubectl get deploy,svc --namespace apps
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/podinfo   1/1     1            1           52m

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/podinfo   ClusterIP   10.96.172.143   <none>        9898/TCP,9999/TCP   52m
```

For more information, refer to the [hnc multi-tenant repo].

[Hierarchical Namespaces]: https://github.com/kubernetes-sigs/hierarchical-namespaces
[Platform Admin]: ../flux2-hnc-multi-tenancy#roles
[hnc plugin]: https://github.com/kubernetes-sigs/hierarchical-namespaces/releases
[onboared the tenant]: ../flux2-hnc-multi-tenancy/tenants/base/dev-team-alpha
[gitrepository for a tenant here]: ../flux2-hnc-multi-tenancy/blob/main/tenants/base/dev-team-alpha/sync.yaml#L9
[hnc multi-tenant repo]: ../flux2-hnc-multi-tenancy
