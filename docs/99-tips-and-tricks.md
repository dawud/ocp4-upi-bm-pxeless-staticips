# Tips and tricks

This section explains some tips and tricks to familiarize with OpenShift 4.

NOTE: [I wrote
this](https://github.com/openshift/training/commit/eab9062e2b5c6a31f510dc2a0e813ee5221e4452)
in the
[openshift/training](https://github.com/openshift/training/blob/master/docs/97-tips-and-tricks.md)
repository back in the day and it still applies.

## NTP configuration

RHCOS uses chronyd to synchronize the system time. The default configuration
uses the `*.rhel.pool.ntp.org` servers:

```bash
$ grep -v -E '^#|^$' /etc/chrony.conf
server 0.rhel.pool.ntp.org iburst
server 1.rhel.pool.ntp.org iburst
server 2.rhel.pool.ntp.org iburst
server 3.rhel.pool.ntp.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
```

As the hosts configuration shouldn't be managed manually, in order to configure
chronyd to use custom servers or a custom setting, it is required to use the
`machine-config-operator` to modify the files used by the masters and workers
by the following procedure:

* Create the proper file with your custom tweaks and encode it as base64:

```bash
cat << EOF | base64
server clock.redhat.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
```

* Create the MachineConfig file with the base64 string from the previous command
as:

```bash
cat << EOF > ./masters-chrony-configuration.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: masters-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,c2VydmVyIGNsb2NrLnJlZGhhdC5jb20gaWJ1cnN0CmRyaWZ0ZmlsZSAvdmFyL2xpYi9jaHJvbnkvZHJpZnQKbWFrZXN0ZXAgMS4wIDMKcnRjc3luYwpsb2dkaXIgL3Zhci9sb2cvY2hyb255Cg==
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/chrony.conf
  osImageURL: ""
EOF
```

Substitute the base64 string with your own.

* Apply it

```bash
oc apply -f ./masters-chrony-configuration.yaml
```

## OCP Master configuration
The master configuration is now stored in a `configMap`. During the installation
process, a few `configMaps` are created, so in order to get the latest:

```bash
oc get cm -n openshift-kube-apiserver | grep config
```

Observe the latest id and then:

```bash
oc get cm -n openshift-kube-apiserver config-ID
```

To get the output in a human-readable form, use:

```bash
oc get cm -n openshift-kube-apiserver config-ID \
  -o jsonpath='{.data.config\.yaml}' | jq
```

For the OpenShift API configuration:

```bash
oc get cm -n openshift-apiserver config -o jsonpath='{.data.config\.yaml}' | jq
```

## Delete 'Completed' pods

During the installation process, a few temporary pods are created. Keeping those
pods as 'Completed' doesn't harm nor waste resources but if you want to delete
them to have only 'running' pods in your environment you can use the following
command:

```bash
oc get pods --all-namespaces | \
  awk '{if ($4 == "Completed") system ("oc delete pod " $2 " -n " $1 )}'
```

## Get pods not running nor completed

A handy one liner to see the pods having issues (such as CrashLoopBackOff):

```bash
oc get pods --all-namespaces | grep -v -E 'Completed|Running'
```

[<< Previous: Verification](14-verification.md) | [Back to README](../README.md)
