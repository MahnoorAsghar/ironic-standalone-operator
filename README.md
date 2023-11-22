# Metal3 Ironic Operator

Operator to maintain an Ironic deployment for Metal3.

## Usage

Let's assume we're creating an Ironic deployment called `ironic` in the
namespace `test`, and that [BMO][bmo] will access it via
`https://ironic.test.svc`.

Start with creating a TLS certificate and a secret for it. In this example,
I'll use a self-signed certificate, you may want to sign it with some
authority, e.g. using built-in Kubernetes facilities.

```bash
openssl req -x509 -new -subj "/CN=ironic.test.svc" -addext "subjectAltName = DNS:ironic.test.svc" \
    -newkey ec -pkeyopt ec_paramgen_curve:secp384r1 -nodes -keyout tls.key -out tls.crt
kubectl create secret tls ironic-tls -n test --key="tls.key" --cert="tls.crt"
```

Now create the Ironic configuration:

```yaml
---
apiVersion: metal3.io/v1alpha1
kind: Ironic
metadata:
  name: ironic
  namespace: test
spec:
  tlsRef:
    name: ironic-tls
  ramdiskSSHKey: "<YOUR SSH PUBLIC KEY HERE>"
```

```
$ kubectl create -f ironic.yaml
```

After some time, you can check the outcome:

```
$ kubectl describe ironic -n test ironic
...
Status:
  Conditions:
    Last Transition Time:  2023-08-25T13:05:35Z
    Message:               ironic is available
    Observed Generation:   2
    Reason:                DeploymentAvailable
    Status:                True
    Type:                  Available
    Last Transition Time:  2023-08-25T13:05:35Z
    Message:               ironic is available
    Observed Generation:   2
    Reason:                DeploymentAvailable
    Status:                False
    Type:                  Progressing
```

Now you can see the service that can be used to access Ironic (it has the same
name as the Ironic object):

```
$ kubectl get service -n test ironic
NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
ironic   ClusterIP   10.96.107.239   <none>        443/TCP   156m
$ # From inside the cluster, ignoring TLS certificates for now:
$ curl -k https://10.96.107.239/
```

The autogenerated credentials secret is linked in the spec:
```
$ IRONIC_SECRET=$(kubectl get ironic -n test ironic -o jsonpath='{.spec.credentialsRef.name}')
$ IRONIC_USER=$(kubectl get secret -n test "$IRONIC_SECRET" -o jsonpath="{.data.username}" | base64 --decode)
$ IRONIC_PASSWORD=$(kubectl get secret -n test "$IRONIC_SECRET" -o jsonpath="{.data.password}" | base64 --decode)
$ # From inside the cluster, accessing an authenticated endpoint:
$ curl -k -u "$IRONIC_USER:$IRONIC_PASSWORD" https://10.96.107.239/v1/drivers
```

By its nature, Ironic listens on a host network and thus is reachable on one of
the Kubernetes nodes, even from the outside (note the port):

```
$ kubectl get pod -n test -l metal3.io/ironic-operator=ironic-service -o wide
NAME                              READY   STATUS    RESTARTS   AGE    IP            NODE                 NOMINATED NODE   READINESS GATES
ironic-service-698899755d-6xzxl   4/4     Running   0          152m   10.89.0.2     kind-control-plane   <none>           <none>
$ curl -k -u "$IRONIC_USER:$IRONIC_PASSWORD" https://10.89.0.2:6385/v1/drivers
```

[bmo]: https://github.com/metal3-io/baremetal-operator

### More detailed example

In this example, a MariaDB database is used instead of SQLite, and a
provisioning network is configured. You will need to generate your TLS
certificates with one more `subjectAltName` in the format `<database
name>-database.<namespace>.svc` (in this example, `ironic-database.test.svc`).
Then another resource needs to be created for the database itself:

```yaml
---
apiVersion: metal3.io/v1alpha1
kind: IronicDatabase
metadata:
  name: ironic
  namespace: test
spec:
  tlsRef:
    name: ironic-tls
---
apiVersion: metal3.io/v1alpha1
kind: Ironic
metadata:
  name: ironic
  namespace: test
spec:
  databaseRef:
    name: ironic
  tlsRef:
    name: ironic-tls
  networking:
    ipAddress: 10.89.0.2
    dhcp:
      networkCIDR: 10.89.0.1/24
      rangeBegin: 10.89.0.10
      rangeEnd: 10.89.0.100
  ramdiskSSHKey: "<YOUR SSH PUBLIC KEY HERE>"
```

This example also shows configuring DHCP for network booting.

## OpenShift notes

- Running a database requires the user to have `nonroot` or similar SCC.
- Running Ironic requires `nonroot` and `hostnetwork`.
