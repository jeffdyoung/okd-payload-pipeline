### Deploy argo-workflow

[temporary workaround]

Create an overlay or directly replace the ${ARGO_FINAL_URL} string with the url the argo-workflows route will be exposed on.
It is argo-server-${NAMESPACE}.apps.${CLUSTER_NAME}.${BASE_DOMAIN} by default.

[/temporary workaround]

```shell
oc apply -k argo-workflows/operator-deployment
```

Patch python image stream to pull entire manifest list.
``` shell
 oc patch is/python -n openshift -p '{"spec":{"tags":[{"name":"3.9-ubi9","importPolicy":{"importMode":"PreserveOriginal"}}]}}'
```

### Deploy the okd workflows and build configs

```shell
oc apply -k variants/fcos-multiarch
```

If taints on the secondary architecture nodes are set, annotate the namespace to allow scheduling builds on those nodes

```shell
oc annotate namespace okd-414 \
  'scheduler.alpha.kubernetes.io/defaultTolerations'='[{"operator": "Exists", "effect": "NoSchedule", "key": "arm64"}]'
```

To run the workflow from the WorkflowTemplate, create a Workflow object like:

```shell
oc create -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
    generateName: builder-build-workflow-
spec:
  arguments:
    parameters:
    - name: architectures
      value: amd64,arm64
    - name: build-config-name
      value: builder
  workflowTemplateRef:
    name: build-multiarch-image
    clusterScope: true
EOF
```

# Build okd 

```shell
oc create -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
    generateName: okd-build
spec:
  arguments:
    parameters:
    - name: architectures
      value: amd64
    - name: cleanup
      value: "true" # deletes any previously built images
    - name: os-image
      value: quay.io/okd/centos-stream-coreos-9:4.12-x86_64
    - name: os-name
      value: centos-stream-coreos-9
    - name: release-image-location 
      value: quay.io/jeffdyoung/release:4.14.0-0.okd-multi
    - name: release-mirror-location
      value: quay.io/jeffdyoung/release
    - name: registry-credentials-secret-ref
      value: registry-robot-token
  workflowTemplateRef:
    name: build-okd
    clusterScope: true
EOF

```

This workflow will import the image at os-image and build the okd release by consuming it.

If, instead, as in the FCOS case, you should build with layering, use the following workflow:

```shell
 oc create -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
    generateName: okd-
spec:
  arguments:
    parameters:
    - name: architectures
      value: arm64
    - name: cleanup
      value: "true"
    - name: os-buildconfig
      value: fedora-coreos
    - name: os-name
      value: fedora-coreos
    - name: release-image-location 
      value: quay.io/jeffdyoung/release:armshift
    - name: release-mirror-location
      value: quay.io/jeffdyoung/release
    - name: registry-credentials-secret-ref
      value: jeffdyoung-openshiftbuild-pull-secret
  workflowTemplateRef:
    name: build-okd
    clusterScope: true
EOF

```

It takes a os-buildconfig value and lacks the os-image one, so that, at the end of the process, it will build the
os content.

## (TODO) Fix Package error for the OKD release
```
+ oc adm release new --registry-config=/tmp/readwrite/.dockerconfigjson --from-image-stream release --insecure=true --mirror quay.io/jeffdyoung/release --to-image quay.io/jeffdyoung/release:4.13.0-0.okd-multi --name=4.13.0-0.okd-multi --keep-manifest-list=true
error: only image streams or releases with public image repositories can be the source for releases when using the default --reference-mode
```


## Publish on quay and handle in the origin release-controller? (TODO)

## Build base images and a list of components (good for testing single build errors)

```shell
 oc create -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
    generateName: okd-component-
spec:
  arguments:
    parameters:
    - name: architectures
      value: amd64,arm64
    - name: cleanup
      value: "true"
    - name: rebuild-base
      value: "false"
    - name: components 
      value: 
        - "baremetal-installer"
        - "cli-artifacts"
  workflowTemplateRef:
    name: build-okd-component
    clusterScope: true
EOF

```