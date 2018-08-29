Gorgeous Code Assessment Test Operations
=======

I decided to go with conventional solution that uses Docker as container and Kubernetes plus Helm as deploying tools.
I wrote down Step by Step manual (although some corrections might be needed).
The assumption is that using Minikube in this case is fine.
Of course, Minikube is usually being used just for testing projects.
In case of the real world project one of alternatives to Minikube can be using AWS and kops.
There is a good manual provided by Amazon how to set it up:
<https://aws.amazon.com/blogs/compute/kubernetes-clusters-aws-kops/>


### Install and Setup Docker
apt install docker.io

For the next step Dockerhub account is needed.

docker login

login with your USERNAME and PASSWORD

create Dockerfile that would look similar to this one:

```
FROM ruby:2.5.0
# Install apt based dependencies required to run Rails as  well as RubyGems.
RUN apt-get update && apt-get install -y \
  build-essential \
  nodejs

# Configure the main working directory.
# RUN mkdir -p /app #this directory already exists so we can comment it out in this case
WORKDIR /app

# Copy the Gemfile as well as the Gemfile.lock and install  the RubyGems.
COPY Gemfile Gemfile.lock ./
RUN gem install bundler && bundle install --jobs 20 --retry 5

# Copy the main application.
COPY . ./

# Expose port 3000 to the Docker host, so we can access it from the outside.
EXPOSE 3000

# The main command to run when the container starts.
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```
Build and test the Image:

docker build -t USERNAME/test-ops .

docker images | grep test-ops

docker push USERNAME/test-ops

### Install Minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.28.2/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

minikube start

### Install Kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/OS_DISTRIBUTION/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl


Check the cluster status:
kubectl cluster-info
kubectl get nodes
kubectl describe node
kubectl get pods
...

### Install and Setup Helm
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh

helm init


Create a chart

helm create test-ops

edit values.yaml

```
# Default values for test-ops.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Here you can change number of replicas
# depending on how many you need
replicaCount: 1

image:
  repository: USERNAME/test-ops
  tag: stable
  pullPolicy: IfNotPresent

nameOverride: ""
fullnameOverride: ""

service:
  name: http
  type: NodePort
  externalPort: 80
  internalPort: 3000

ingress:
  enabled: false
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  path: /
  hosts:
    - chart-example.local
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #  cpu: 100m
  #  memory: 128Mi
  # requests:
  #  cpu: 100m
  #  memory: 128Mi

nodeSelector: {}

tolerations: []

affinity: {}
```
	
edit templates/deployment.yaml

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: {{ include "test-ops.fullname" . }}
  labels:
    app: {{ include "test-ops.name" . }}
    chart: {{ include "test-ops.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "test-ops.name" . }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ include "test-ops.name" . }}
        release: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.internalPort }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
{{ toYaml .Values.resources | indent 12 }}
    {{- with .Values.nodeSelector }}
      nodeSelector:
{{ toYaml . | indent 8 }}
    {{- end }}
    {{- with .Values.affinity }}
      affinity:
{{ toYaml . | indent 8 }}
    {{- end }}
    {{- with .Values.tolerations }}
      tolerations:
{{ toYaml . | indent 8 }}
    {{- end }}
```
	
helm dep list
helm dep update .
helm lint .

helm repo update

helm install tech-ops

### And Fire it Up! :-)
