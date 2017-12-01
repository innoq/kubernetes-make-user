## What does this do?

This allows you to create users for your Kubernetes cluster.

## How is it done?

This is mostly a script around the [instructions available at
bitnami](https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/)
and in the
[one](https://kubernetes.io/docs/admin/authentication/#x509-client-certs)
or
[another](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)
part of the official documentation.

## How do I install this?

### Direct way

Have `openssl` and `kubectl` and `python3` (3.5 or newer) installed on
your machine and copy `src/kubernetes-make-user` to somewhere in your
`$PATH`.

### Container way (not yet implemented)

Assuming you have Docker installed already, run something like

    cd src && docker build -t local-only/kubernetes-make-user:latest .

(There probably is no Docker registry at `local-only`, but that's ok
as long as you don't try to interact with it.  Just neither `docker
push` nor `docker pull` this image.)

You can then need to use `docker` to run `kubernetes-make-user`.
Something like the following should work:


```
docker run -ti --rm -v "$(pwd):/data" local-only/kubernetes-make-user:latest \
      --user="$(id -u):$(id -g)" kubernetes-make-user ...
```

The `--user="$(id -u):$(id -g)"` stance is useful so the generated
files belong to the user running the command, rather than `root`.  It
serves no other purpose and you can leave it out if there is no `id`
command on your system, or the `docker` command is typed in to a shell
that does not understand `$(...)`.  In the latter case, you may also
want to replace the `$(pwd)` part with a (manually typed-in) absolute
directory information.

## How do I become admin to run this?

The script expects a file `ca.key` that contains the private key of
the CA used by the cluster.  Whoever has that file is cluster-admin.

You can typically harvest that from the docker container that runs
your cluster's api server.

Here's instructions that work in a KOPS-installed Kubernetes 1.7
cluster:

Run

    kubectl get -n kube-system po

From the output, grab the precise name of the (or, if you have
several, of one) `kube-apiserver` POD.

Use that to obtain the private CA's (certificate authority) private key:

    kubectl exec -n kube-system kube-apiserver-... cat /srv/kubernetes/ca.key > ca.key

**Whoever can read `ca.key` owns your cluster.**  So be careful what
you do with that file.

Once at it, also grab the private CA's public key with

    kubectl exec -n kube-system kube-apiserver-... cat /srv/kubernetes/ca.crt > ca.crt

### I could not obtain the CA's private key that way.

If these instructions don't work for you, maybe you aren't the cluster
admin?

If you *are* the legitimate cluster admin, look at the precise command
line of the API server

    kubectl get -n kube-system po/kube-apiserver-... -o jsonpath='{.spec.containers[*].command[*]}'

and find the `--client-ca-file` switch.  This gives you the location
of the CA's public key.

The private key may be sitting in the same directory.  If it isn't,
I'm afraid you may need to find the software or person that generated
that public CA key.  When they did that, they also had the private
one.  It is to be hoped they kept it and can hand it out to you.

If there is no `--client-ca-file` switch, maybe your cluster is
running without CA?  See the
[documentation](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)
how to create one.

## How do I run this?

Short version:

Have the cluster CA file `ca.key` in the current directory and run

    kubernetes-make-user --user=<user> --group=<group1> --group=<group2> --days=60

This produces a client key file `<user>.key` and a client certificate
file `<user>.crt`.

If you have a mature user, hand both of these to the user.  The author
recommends to *transport the `<user>.key` file via secure mechanism.*

### My user wants to generate their own key and just hand me a CSR.

That's a cool idea, as it removes the need for a secure communication
channel between you and your user.  _Information how to proceed to be
filled in later._

## How do these files make the intended person a cluster user?

To start, you should have the cluster safely tugged into your
configuration, through something like


```
kubectl config set-context myuser-in-mycluster --cluster=mycluster --user=myuser
kubectl config set-cluster mycluster --embed-certs=true --server=https://api.mycluster.example.org --certificate-authority=ca.crt 
kubectl config use-context myuser-in-mycluster
```

That done, run the crucial

```
kubectl config set-credentials myuser --embed-certs=true --client-certificate=myuser.crt --client-key=myuser.key 
```

You can do all of the above on top of an empty `--kubeconfig` file and
hand that file to the user.  This may be a good approach dealing with
a first-time Kubernetes user.
