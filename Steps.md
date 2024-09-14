replicated login

replicated app create Harbor

export REPLICATED_APP=harbor-haddock

helm pull --untar oci://registry-1.docker.io/bitnamicharts/harbor

# Update Chart.yaml with this dependency
dependencies:
- name: replicated
  repository: oci://registry.replicated.com/library
  version: 1.0.0-beta.27

helm package . --dependency-update

mkdir manifests

mv harbor-23.0.3.tgz manifests

cd manifests

touch harbor.yaml kots-app.yaml k8s-app.yaml embedded-cluster.yaml

# harbor.yaml
apiVersion: kots.io/v1beta2
kind: HelmChart
metadata:
  name: harbor
spec:
  # chart identifies a matching chart from a .tgz
  chart:
    name: harbor
    chartVersion: 2.11.1
  optionalValues:
  - when: 'repl{{ eq Distribution "embedded-cluster" }}'
    recursiveMerge: false
    values:
      service:
        type: NodePort
        nodePorts:
          http: "32000"

# kots-app.yaml
apiVersion: kots.io/v1beta1
kind: Application
metadata:
  name: harbor
spec:
  title: Harbor
  statusInformers:
    - deployment/harbor
  ports:
    - serviceName: "harbor"
      servicePort: 3000
      localPort: 32000
      applicationUrl: "http://harbor"
  icon: https://raw.githubusercontent.com/cncf/artwork/master/projects/kubernetes/icon/color/kubernetes-icon-color.png

# k8s-app.yaml
apiVersion: app.k8s.io/v1beta1
kind: Application
metadata:
  name: "harbor"
spec:
  descriptor:
    links:
      - description: Open App
        # needs to match applicationUrl in kots-app.yaml
        url: "http://harbor"

# embedded-cluster.yaml
apiVersion: embeddedcluster.replicated.com/v1beta1
kind: Config
spec:
  version: 1.9.1+k8s-1.29

replicated release lint --yaml-dir .

# Create a release
replicated release create --yaml-dir .