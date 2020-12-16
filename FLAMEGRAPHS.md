These are my notes to get a Flamegraph out of a Openshift container:

## Let the perf sidecar be injected into the pods:

Follow https://developer.ibm.com/recipes/tutorials/profiling-applications-deployed-on-kubernetes-with-sidecar-injector/
I had to change deployment apiVersion from `extensions/v1beta1` to `apps/v1` and add this selector:

```yaml
  selector:
    matchLabels:
      app: perf-sidecar-injector
```

Change port from 443 to 8443 in both the pod and service. Use emptyDir as the mount for `/out`. Change args to trigger the perf manually instead of immediatelly when the container starts.
```yaml
args:
          - 'record'
          - '1'
          - '60'
          - '1'
          - '-g'
```

The version of `perf` used in the default image had some defects (e.g. segfaults when trying to produce a system-wide recording). I have replaced that with `fedora:rawhide` with manually installed `perf`.

In the `mutatingwebhookconfiguration` use following labels to select the pods:

```yaml
namespaceSelector:
  matchLabels:
    perf-sidecar-injector-webhook/inject: "yes"
objectSelector:
  matchLabels:
    perf-sidecar-injector-webhook/inject: "yes"
```
(suggested filtering based on annotations didn't work for me)

Now you can `oc rsh -c perf-sidecar --pod-- touch /tmp/startperf` and a file appears in /out. Copy it out from the container to `/tmp/perf.data`.

## Mount the image

To be able to resolve symbols correctly, run the image in podman and mount it

```
podman run -d --rm --name my-image --entrypoint tail --image-- -f /dev/null
MNT=$(podman mount my-image)
```

You also need to get the kallsyms from the Openshift container (`oc cp` does not work here):
```
oc exec --pod-- -- cat /proc/kallsyms > /tmp/kallsyms
```

And now you can run perf script:
```
perf script -i /tmp/perf.data --symfs=$MNT --kallsyms=/tmp/kallsyms -v > /tmp/perf.script
git clone https://github.com/brendangregg/FlameGraph
FlameGraph/stackcollapse-perf.pl /tmp/perf.script > /tmp/perf.collapsed
FlameGraph/flamegraph.pl --width 1800 /tmp/perf.collapsed > /tmp/flames.svg
```