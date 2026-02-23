# postgresql-spiffe-demo

Demonstration of using Red Hat Zero Trust Workload Identity Manager to secure and communicate with a PostgreSQL instance.

## Prerequisites

Ensure the following constraints are met before moving forward:

* OpenShift Cluster with elevated permissions
* Zero Trust Workload Identity Manager (ZTWIM) Installed
    * Installed CRDs with the name `zero-trust-workload-identity-manager`

## Build spife-helper custom image (Optional)

When the [spiffe-helper](https://github.com/spiffe/spiffe-helper) issues new certificates for the PostgreSQL instance, it will reload the PostgreSQL configuration so that it utilizes the latest generated certificates. A custom spiffe-helper image is required that adds the PostgreSQL client tools so that it can before the refresh process. An existing image is provided. However, if desired, a custom image can be built and utilized.

### Build the custom image

Execute the following command to build the custom image

```shell
podman build -f spiffe-helper-postgresql/Containerfile -t localhost/spiffe-helper-postgresql spiffe-helper-postgresql
```

Push the image to a registry of your choosing.

Update the reference to the image in the [postgresql-spiffe.yaml](deploy/postgresql-spiffe.yaml) resource.

## Deploy the Resources to OpenShift

Apply the PostgreSQL server and client to the OpenShift cluster

```shell
oc apply -f deploy/postgresql-spiffe.yaml
oc apply -f deploy/postgresql-spiffe-client.yaml
```

## Investigate the Deployment

Two namespaces will be created: `postgresql-spiffe` and `postgresql-spiffe-client` for the PostgreSQL instance and a sample client accordingly. 

Confirm pods are running in both namespaces:

```shell
oc get pods -n postgresql-spiffe
oc get pods -n postgresql-spiffe-client
```

A x509 certificate has been retrieved by the _spiffe-helper_ containers in each pod.

First, extract the certificate from the PostgreSQL pod:

```shell
oc rsh -n postgresql-spiffe -c postgresql-spiffe deployment/postgresql-spiffe cat /opt/postgresql-certs/svid.pem | openssl x509 -noout -text
```

Notice the `CN` field includes the hostname that will be used by clients connecting to the instance. This configuration is defined within the _postgresql-spiffe_ `ClusterSPIFFEID`.

A similar configuration is defined on the client X509 certificate:

```shell
oc rsh -n postgresql-spiffe-client -c postgresql-spiffe-client deployment/postgresql-spiffe-client cat /opt/postgresql-certs/svid.pem | openssl x509 -noout -text
```

Instead of the hostname, the PostgreSQL username `postgresql_spiffe` is specified as the CN as defied on the _postgresql-spiffe-client_ `ClusterSPIFFEID`.

# Verify the solution

Finally, verify the solution.

Start a remote shell session with the PostgreSQL client instance to initiate a session with the PostgreSQL instance using mTLS.

```shell
oc rsh -n postgresql-spiffe-client -c postgresql-spiffe-client deployment/postgresql-spiffe-client
```

Connect to the PostgreSQL instance using SPIFFE provided certificates

```shell
psql "host=postgresql-spiffe.postgresql-spiffe.svc port=5432 user=postgresql_spiffe dbname=testdb sslmode=verify-full sslcert=/opt/postgresql-certs/svid.pem sslkey=/opt/postgresql-certs/svid.key sslrootcert=/opt/postgresql-certs/svid_bundle.pem"
```

A successful connection should be established.

Feel free to perform queries as you see fit against the database.

Access to this instance can only be established using the SPIFFE provided certificates securing the instance with zero trust considerations in mind.

## Future Considerations

The assets in this repository are created without persistent storage, which can be easily added if necessary to the `/var/lib/pgsql/data` directory of the PostgreSQL _Deployment_ if desired. 