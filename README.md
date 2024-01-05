# Kubernetes Debian Mirror Template

This is a template deployment to self-host a debian mirror. Good for home labs, enterprises and everything in-between.

## Requirements

- A deployed kubernetes cluster and kubectl configured.
  - This guide does not go through setting up a kubernetes cluster
  - If you don't have kubernetes deployed but have a cluster and ansible skills, check out my [HomelabKubernetes repository](https://github.com/nuvious/HomelabKubernetes)
  to deploy k3s or microk8s to your target nodes easily.
- Some persistent storage resource such as an nfs, ceph, etc
  - A debian mirror requires a lot of persistent storage
  - Template uses hostPath and should be replaced with a proper network storage solution
- Basic Kubernetes Knowledge
  - While we will provide guides on patching the deployment we will not go into minute detail on individual resources
  - Links are provided where appropriate in this guide to link out to resource documentation
  - For documentation on different resources used in this project, check out [kubernetes documentation](https://kubernetes.io/docs)

## Project Structure

This project follows the guidance outlined in [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#bases-and-overlays)
and uses bases and overlays to define deployments. Deployment-specific configurations should not be included in the
bases, and overlays should be generated with patches to the base to cater to specific deployments. Below are the
files in the project structure and their purpose:

- base
  - conf
    - mirror.list
      - A list of mirrors in [source.list syntax](https://wiki.debian.org/SourcesList) that defines the mirrors, distros
        and components desired for mirroring.
    - nginx.conf
      - A definition for the persistent http service that will remain up between debmirror sync jobs.
  - debmirror-mirror-job.yaml
    - Job that periodically runs debmirror with the prescribed mirror list to update available packages.
  - nginx.yaml
    - The persistent http service.
  - namespace.yaml
    - A namespace resource; defaults to `debmirror`.
  - service.yaml
    - Defines the http service with the namespace.
  - ingress.yaml
    - Defines the ingress for the http service.
  - kustomization.yaml
    - The base kustomization configuration.

## Quick Start

We will be creating an overlay for this specific deployment. In this overlay we can add deployment specific resources
such as secrets or patching base resources. If you are unfamiliar with patching in a kustomize configuration using
overlays, please review [patching documentation](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#customizing)
maintained by kubernetes.

### Create Initial Overlay Directory

First we need to create an overlay. To do that simply make a directory like so:

```bash
mkdir -p overlays/local
```

Replace `local` with any meaningful directory name for your deployment.

### Patch Persistent Storage

The [debmirror](https://linux.die.net/man/1/debmirror) tool used in the base container requires persistent storage;
[as much as 100 gigs](https://help.ubuntu.com/community/Debmirror) if mirroring source and binaries. In this example
I will be patching the [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) volume type defined
in the nginx deployment and the debmirror job. HostPath is the default one defined in the base as a placeholder but is
only okay to use if you're on a single-node kubernetes deployment. For multi-node an [nfs](https://kubernetes.io/docs/concepts/storage/volumes/#nfs),
[ceph](https://kubernetes.io/docs/concepts/storage/volumes/#cephfs) or other network based storage should be used.

For this example I will be using a cephfs solution without a persistent volume claim but consult kubernetes
documentation on [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) and the lifecycle
of [Persistent Volume Claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#lifecycle-of-a-volume-and-claim)
for best-practices guidance. Since I'm using cephfs I will need to define a secret with the key used to authenticate
with ceph storage area network:

```bash
cat > overlays/local/ceph-secret.yaml << EOF
apiVersion: v1
stringData:
  key: [INSERT YOUR CEPHFS KEY]
kind: Secret
metadata:
  name: ceph-secret
EOF
```

Now let's create the nginx deployment storage patch:

```bash
cat > overlays/local/nginx-patch.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: debmirror
    component: http
  name: debmirror-nginx
spec:
  template:
    spec:
      containers:
      - name: debian-mirror-nginx
        volumeMounts:
        - name: data-patched
          mountPath: /usr/share/nginx/html
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
      volumes:
      - name: mirror-list
        configMap:
          name: mirror-list
      - name: nginx-conf
        configMap:
          name: nginx-conf
      - name: data-patched
        cephfs:
          monitors:
            - 192.168.1.70:6789
            - 192.168.1.71:6789
            - 192.168.1.72:6789
          user: kube
          secretRef:
            name: ceph-secret
          path: /kube/debmirror/data
EOF
```

Next we'll modify the debmirror job to point to the same persistent storage. Note that instead of
`[ROOT PERSISTENCE DIR]/data` we reference `[ROOT PERSISTENCE DIR]` without the `/data` subdirectory. That's because
the data directory is all that's needed to serve on the http service and the other directories are used by debmirror
only for keeping track of the state of the mirror.

Now is also a good time to modify the
[CronJob's schedule](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#writing-a-cronjob-spec)
if desired.

```bash
cat > overlays/local/debmirror-job-patch.yaml << EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: debmirror-cron
spec:
  schedule: "0 3 * * 3"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: debmirror-cron
            volumeMounts:
            - name: data-patched
              mountPath: /var/spool/apt-mirror
            - name: mirror-list-patched
              mountPath: /etc/apt/mirror.list
              subPath: mirror.list
          volumes:
          - name: mirror-list-patched
            configMap:
              name: mirror-list
          - name: data-patched
            cephfs:
              monitors:
                - 192.168.1.70:6789
                - 192.168.1.71:6789
                - 192.168.1.72:6789
              user: kube
              secretRef:
                name: ceph-secret
              path: /kube/debmirror # DO NOT USE /data SUBDIRECTORY HERE!
EOF
```

### Modify Configurations and Ingress (Optional)

We will be replacing configmap resources defined in the base. This is only required if you want your hostname to be
something besides `localhost` and/or if you want to pull updates from mirrors other than [debian.org](http://ftp.us.debian.org/debian/).

First make the configuration directory:

```bash
mkdir -p overlays/local/conf
```

Next if it's desired to change the resolving hostname, create a `default.conf` with the desired hostname in
[nginx configuration native syntax](https://nginx.org/en/docs/).

```bash
cat > overlays/local/conf/default.conf << EOF
server {
    listen       80;
    listen  [::]:80;
    server_name  debmirror.local;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        autoindex on;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
EOF
```

Next create a patch for the ingress to replace the hostname with the desired hostname:

```bash
cat > overlays/local/ingress-patch.yaml << EOF
- op: replace
  path: /spec/rules/0/host
  value: debmirror.local
EOF
```

Finally if it's desired to modify the mirror list, create a `mirror.list` replacement. I live close to Indiana so I
changed mine to the Purdue debian mirror. I also added `armhf` and `arm64` architectures to the mirror so I could update
my standard Debian deployments as well as my raspberry pi based deployments.

```bash
cat > overlays/local/conf/mirror.list << EOF
set base_path         /var/spool/apt-mirror
set mirror_path       $base_path/data
set skel_path         $base_path/skel
set var_path          $base_path/var
set postmirror_script $var_path/postmirror.sh
set defaultarch       amd64
set run_postmirror    0
set nthreads          6
set limit_rate        100m
set _tilde            0
set unlink            1
set use_proxy         off
set http_proxy        127.0.0.1:3128
set proxy_user        user
set proxy_password    passwordx
deb http://plug-mirror.rcac.purdue.edu/debian stable main contrib non-free-firmware
deb [arch=armhf] http://plug-mirror.rcac.purdue.edu/debian stable main contrib non-free-firmware
deb [arch=arm64] http://plug-mirror.rcac.purdue.edu/debian stable main contrib non-free-firmware
deb-src http://plug-mirror.rcac.purdue.edu/debian stable main contrib non-free-firmware
deb http://security.debian.org/debian-security stable-security main contrib non-free-firmware
deb [arch=armhf] http://security.debian.org/debian-security stable-security main contrib non-free-firmware
deb [arch=arm64] http://security.debian.org/debian-security stable-security main contrib non-free-firmware
deb-src http://security.debian.org/debian-security stable-security main contrib non-free-firmware
deb http://plug-mirror.rcac.purdue.edu/debian stable-updates main
deb [arch=armhf] http://plug-mirror.rcac.purdue.edu/debian stable-updates main
deb [arch=arm64] http://plug-mirror.rcac.purdue.edu/debian stable-updates main
deb-src http://plug-mirror.rcac.purdue.edu/debian stable-updates main contrib non-free-firmware
deb http://plug-mirror.rcac.purdue.edu/debian stable-backports main
deb [arch=armhf] http://plug-mirror.rcac.purdue.edu/debian stable-backports main
deb [arch=arm64] http://plug-mirror.rcac.purdue.edu/debian stable-backports main
deb-src http://plug-mirror.rcac.purdue.edu/debian stable-backports main contrib non-free-firmware
deb [arch=armhf] http://plug-mirror.rcac.purdue.edu/raspbian stable main contrib non-free firmware rpi
EOF
```

### Other Patches

One can also change the namespace deployed simply by modifying the namespace.yaml resource and/or modifying the
kustomization.yaml. Changing the deployed namespace is not covered in this tutorial, but if desired, refer to the
[kustomization documentation](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization) maintained
by kubernetes for guidance on making these or other additional changes in the overlay.

### Kustomize and Deploy

All that's left is to tie together all of the above with a `kustomization.yaml` file and deploy it. Below is the basic
template with optional line items highlighted with comments:

```bash
cat > overlays/local/kustomization.yaml << EOF
namespace: debmirror
resources:
  - ../../base
  - ceph-secret.yaml
configMapGenerator:
  - name: mirror-list # Only necessary if you updated mirror.list
    behavior: replace
    files:
      - conf/mirror.list
  - name: nginx-conf # Only necessary if you overrode the default hostname localhost
    behavior: replace
    files:
      - conf/default.conf
patches:
  - target:
      kind: Ingress
    path: ingress-patch.yaml # Only necessary if you overrode the default hostname localhost
patchesStrategicMerge:
  - debmirror-job-patch.yaml
  - nginx-patch.yaml
EOF
```

Now simply apply the overlay. The below command will also execute a watch on `kubectl get all -n debmirror` so you
can monitor the deployment:

```bash
kubectl apply -k overlays/local && \
    watch kubectl get all -n debmirror
```

The output should resemble the following:

```plaintext
Every 2.0s: kubectl get all -n debmirror     my-workstation: Fri Jan  5 14:17:23 2024

NAME                                   READY   STATUS    RESTARTS   AGE
pod/debmirror-nginx-85ff7b64fc-jpcts   1/1     Running   0          46s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/debmirror-service   ClusterIP   10.152.183.199   <none>        80/TCP    47s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/debmirror-nginx   1/1     1            1           47s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/debmirror-nginx-85ff7b64fc   1         1         1       47s

NAME                            SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/debmirror-cron   0 3 * * 3   False     0        <none>          47s
```

### Force a Sync Job Execution

Initiate a synchronization job manually with the following command:

```bash
kubectl create job -n debmirror  --from=cronjob/debmirror-cron debmirror-manual-job
```

You can then monitor the job by watching the logs of the generated pod:

```bash
# Run kubectl get pods to get the name of you pod
kubectl logs -f -n debmirror pods/debmirror-manual-job-86lbn 
```

You should see output similar to the below:

```plaintext
Downloading 318 index files using 6 threads...
Begin time: Fri Jan  5 19:27:21 2024
[6]... [5]... [4]... [3]... [2]... [1]... [0]... 
End time: Fri Jan  5 19:27:34 2024

Processing translation indexes: [TTTTTTTTTTTTT]
...
```

When done, be sure to cleanup this resource with `kubectl delete job/debmirror-manual-job -n debmirror`.

The initial sync will take a long time and download 100's of GB's depending on your mirror list configuration. For me
since I was downloading x86-64, armhf and arm64, it was 380+ GB's. Follow on mirror runs will need far less. After your
initial sync job wait for the next scheduled sync job and you should see a much smaller size of download logged. Here's
logs from my first and most recent sync for comparison:

```bash
356.0 GiB will be downloaded into archive.
Downloading 304327 archive files using 6 threads...
...
```

```bash
2.6 GiB will be downloaded into archive.
Downloading 287 archive files using 6 threads...
...
```

Notice the 2.6 GB vs several hundred. That shows your mirror is working and updating as expected. You can also
inspect the completions with `kubectl get jobs -n debmirror`:

```bash
$> kubectl get jobs -n debmirror
NAME                       COMPLETIONS   DURATION   AGE
20231228-apt-mirror-job    1/1           4h18m      8d
apt-mirror-cron-28404480   1/1           8m44s      2d11h
```

You can use a browser or `curl` to inspect the file structure:

```bash
$> curl debmirror.local
<html>
<head><title>Index of /</title></head>
<body>
<h1>Index of /</h1><hr><pre><a href="../">../</a>
<a href="plug-mirror.rcac.purdue.edu/">plug-mirror.rcac.purdue.edu/</a>                       28-Dec-2023 17:14                   -
<a href="security.debian.org/">security.debian.org/</a>                               28-Dec-2023 21:03                   -
</pre><hr></body>
</html>
$>
```

### Update `sources.list`

After your mirror is synced you should be able to point your package repository to the new mirror. Here's an example
`/etc/apt/sources.list` I have from my debian template VM I used in my cluster:

```plaintext
deb http://debmirror..local/plug-mirror.rcac.purdue.edu/debian stable main contrib non-free-firmware
deb-src http://debmirror..local/plug-mirror.rcac.purdue.edu/debian stable main contrib non-free-firmware

deb http://debmirror..local/plug-mirror.rcac.purdue.edu/debian stable-updates main contrib non-free-firmware
deb-src http://debmirror..local/plug-mirror.rcac.purdue.edu/debian stable-updates main contrib non-free-firmware
```

Your specific configuration will vary based on the mirrors you use, the distros/architectures you select and the
components you select. Also you can modify the paths in your nginx service and/or ingress as desired if you're only
mirroring from a single mirror to exclude the hostname directory paths (ex make
`deb http://debmirror..local/plug-mirror.rcac.purdue.edu/debian` just
`deb http://debmirror..local/debian`).

After updating, your `sources.list` you should be able to run `apt update` and see your server pull from your new local
repository:

```bash
nuvious@debiantemplate:~$ sudo apt update
Hit:1 http://debmirror.local/plug-mirror.rcac.purdue.edu/debian stable InRelease
Get:2 http://debmirror.local/plug-mirror.rcac.purdue.edu/debian stable-updates InRelease [52.1 kB]
Fetched 52.1 kB in 0s (128 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
nuvious@debiantemplate:~$
```

## Contributing

All contributions are welcome and MUST conform to the [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
documentation.

The base image of this project will also persist as the one I maintain under `nuvious/apt-mirror-docker`. If you would
like to contribute to the image used in this deployment please check out
[nuvious/apt-mirror-docker github repository](https://github.com/nuvious/apt-mirror-docker) and file issues/make
contributions there.
