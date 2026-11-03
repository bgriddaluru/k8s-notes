Kubelet turns the ConfigMap or Secret into files on the node, then the runtime mounts that directory into the container. Updates replace the files—not the container’s mount path.

Take a ConfigMap with one key,  app.conf , mounted at  /etc/config . Here’s the normal Linux, whole-directory mount flow.

1. Fetch the data and write files on the node

Kubelet registers the Pod’s dependency on the ConfigMap. The volume plugin retrieves its contents and writes each selected key as a file.

With the default kubelet root directory, the volume lives at:

/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~configmap/<volume-name>/

A Secret uses  kubernetes.io~secret  instead. ConfigMap files are normally disk-backed; Secret volumes use memory-backed  tmpfs  on Linux. Secret files contain the actual decoded bytes, not the API’s base64 representation.

2. Mount that directory into the container

The runtime exposes the node-side directory at the requested container path, normally read-only:

Node volume directory  →  container /etc/config

The application reads  /etc/config/app.conf  like any other file. It does not contact Kubernetes.

3. Detect changes, then refresh the files

With watch-based change detection, kubelet watches the referenced object through the API server and keeps its latest contents in a local cache.

The watch updates the cache; it does not immediately rewrite the volume. Periodic Pod reconciliation asks the volume plugin to refresh, which reads the latest cached contents and writes them if changed. Consequently, updates are eventual, not instantaneous. Other configured strategies use cached or direct API reads instead of watches.

4. Switch file contents without changing the mount

Kubelet uses a directory-and-symlink layout:

```
/etc/config/
  app.conf → ..data/app.conf
  ..data   → ..version-1/
  ..version-1/
    app.conf
```

For an update, it writes a complete new directory, then atomically switches  ..data :

```
app.conf → ..data/app.conf
..data   → ..version-2/
```

The mount stays at  /etc/config ; reopening  /etc/config/app.conf  now reads the new version. Kubelet does not overwrite the old file in place or restart the container.

Two important consequences:

• An application holding an already-open file can continue reading the old version; it must reopen/reload to use the new contents.
• A file mounted using  subPath  does not receive these automatic updates.

