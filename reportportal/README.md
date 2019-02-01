# k8s
Kubernetes/Helm configs for ReportPortal

**Do note that this is a Beta version**


This Helm project is created to setup ReportPortal with only one commando.  
The help chart is tested on Minikube and creates a fully working ReportPortal project

The chart installs all mandatory services to run ReportPortal

The Helm chart installation consist of the following .yaml files:

- Statefulset and Service files of: `Migrations, Api, Index, UAT, UI, RabbitMq, PostgreSQL` that are used for deployment and communication between services.
- PersistentVolume files: `PostgreSQL`
- A `Ingress` to access the UI
- A `values.yaml` which exposes a few of the configuration options in the
charts.
- A `templates/_helpers.tpl` file which contains helper templates. 

Start to minikube with options:
- `minikube --memory 4096 --cpus 2 start`

Installation of ingress plugin:
- `minikube addons enable ingress`

For deploy Helm Chart, need to initialization Helm:
- `helm init`

You can deploy this chart with `helm install ./<project folder>`. 

Once it's installed please make sure that the PersistentVolumes directories are created

Commando to create those folders:
- `minikube ssh`
- `sudo mkdir /mnt/data/db`
- `sudo mkdir /mnt/data/console`
- `sudo mkdir /mnt/data/elastic`

Also make sure that the vm.max_map_count is setup
- `minikube ssh`
- `sudo sysctl -w vm.max_map_count=262144`

The url to reach ReportPortal is http://reportportal.k8.com
Make sure that the url is added in the host file and the ip is the K8 ip address
Commando to get ip adress of minikube:
- `minikube ip`
Example for host file:
- `192.168.99.100	reportportal.k8.com`

Variables is presents in value.yml. Report Portal use next images in variables:

- traefik: traefik 1.6 
- serviceindex:  reportportal/service-index 4.0.0
- consul: consul 1.0.6
- serviceauthorization: reportportal/service-authorization 4.1.0
- serviceui: reportportal/service-ui 4.1.0
- serviceanalyzer: reportportal/service-analyzer 4.1.0
- serviceapi:  reportportal/service-api 4.1.0

Requirements:
- `RabbitMq`
- `PostgreSQL`

Before you deploy reportportal you should have installed requirements. Versions are described in requirements.yaml.

Also you should specify correct postgresql and rabbitmq addresses and ports in values.yaml. Also it could be an external existing installation:
```
postgresql:
  SecretName: ""
  installdep:
    enable: false
  endpoint:
    external: true
    address: db-postgresql.default.svc.cluster.local
    port: 5432
    user: postgres

rabbitmq:
  SecretName: ""
  installdep:
    enable: false
  endpoint: 
    external: true
    address: mq-rabbitmq-ha.default.svc.cluster.local
    port: 5672
    user: rabbitmq
    apiport: 15672
    apiuser: rabbitmq
```

### Installation notes

#### Minikube
1. Make sure you have Kubernetes up and running
2. Reportportal requires installed [postgresql](https://github.com/helm/charts/tree/master/stable/postgresql) and [rabbitmq](https://github.com/helm/charts/tree/master/stable/rabbitmq-ha) to run. Required versions of helm charts are described in requirements.yaml
If you don't have your own postgresql and rabbitmq instances, they can be installed from official helm charts. 

For example to install postgresql please use this commands:
```sh
helm dependency build ./reportportal/
helm install --name <postgresql_chart_name> --set postgresqlDatabase=reportportal ./reportportal/charts/postgresql-3.9.1.tgz
```
Once PostgreSql has been deployed, copy address and port from output notes. Should be something like this:
```
** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS name from within your cluster:

    <postgresql_chart_name>-postgresql.default.svc.cluster.local - Read/Write connection
To get the password for "postgres" run:

    export POSTGRESQL_PASSWORD=$(kubectl get secret --namespace default <postgresql_chart_name>-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

To connect to your database run the following command:

    kubectl run <postgresql_chart_name>-postgresql-client --rm --tty -i --restart='Never' --namespace default --image bitnami/postgresql --env="PGPASSWORD=$POSTGRESQL_PASSWORD" --command -- psql --host <postgresql_chart_name>-postgresql -U postgres



To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/<postgresql_chart_name>-postgresql 5432:5432 &
    psql --host 127.0.0.1 -U postgres
```
RabbitMq chart can be installed in the same manner:
```sh
helm install --name <rabbitmq_chart_name> --set rabbitmqUsername=rabbitmq,rabbitmqPassword=<rmq_password>  ./reportportal/charts/rabbitmq-ha-1.18.0.tgz
```

Once RabbitMQ has been deployed, copy address and port from output notes. Should be something like this:

```
** Please be patient while the chart is being deployed **

  Credentials:

    Username      : rabbitmq
    Password      : $(kubectl get secret --namespace default <rabbitmq_chart_name>-rabbitmq-ha -o jsonpath="{.data.rabbitmq-password}" | base64 --decode)
    ErLang Cookie : $(kubectl get secret --namespace default <rabbitmq_chart_name>-rabbitmq-ha -o jsonpath="{.data.rabbitmq-erlang-cookie}" | base64 --decode)


  RabbitMQ can be accessed within the cluster on port 5672 at <rabbitmq_chart_name>-rabbitmq-ha.default.svc.cluster.local

  To access the cluster externally execute the following commands:

    export POD_NAME=$(kubectl get pods --namespace default -l "app=rabbitmq-ha" -o jsonpath="{.items[0].metadata.name}")
    kubectl port-forward $POD_NAME --namespace default 5672:5672 15672:15672

  To Access the RabbitMQ AMQP port:

    amqp://127.0.0.1:5672/

  To Access the RabbitMQ Management interface:

    URL : http://127.0.0.1:15672
```

3. After postgresql and rabbitmq are up and running, edit values.yaml to adjust ReportPortal settings.
Insert real values of postgresql and rabbitmq addresses and ports:
```
postgresql:
  SecretName: ""
  installdep:
    enable: false
  endpoint:
    external: true
    address: db-postgresql.default.svc.cluster.local
    port: 5432
    user: postgres

rabbitmq:
  SecretName: ""
  installdep:
    enable: false
  endpoint: 
    external: true
    address: mq-rabbitmq-ha.default.svc.cluster.local
    port: 5672
    user: rabbitmq
    apiport: 15672
    apiuser: rabbitmq
```
Adjust resources for each pod if needed:
```
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 250m
      memory: 512Mi
```
Set ingress controller configuration for UI like this:
```
# ingress configuration for the ui
ingress:
  hosts:
    - reportportal.k8.com
```
4. Once values.yaml is adjusted, helm package can be created and deployed by executing:
```sh
helm package ./reportportal/
helm install --name <reportportal_chart_name> --set postgresql.SecretName=<db_chart_name>-postgresql,rabbitmq.SecretName=<rabbitmq_chart_name>-rabbitmq-ha ./reportportal-5.0.0.tgz
```
Once deployed, you can validate application is up and running by opening your ingress address server or NodePort:
```example
gateway     NodePort   10.233.48.187  <none>       80:31826/TCP,8080:31135/TCP  2s
```
5. Open in browser http://10.233.48.187:8080 page. Defalut login and password is:
```
default
1q2w3e
```
P.S: If you can't login - please check logs of api and uat pods. It take some time to initialize.

#### GKE

1. To deploy Report Portal to the GKE you should have Kubernetes cluster up and running

    1.1 Setup Helm on the GKE:
   ```
   helm init
   kubectl create serviceaccount --namespace kube-system tiller
   kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
   kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
   helm init --service-account tiller --upgrade
   ```

2. To start deploy application to the GKE use 2 and points from "Minikube" section regarding;

3. Helm package can be created and deployed by executing:
   ```sh
   helm package ./reportportal/
   helm install --name <reportportal_chart_name> --set postgresql.SecretName=<db_chart_name>-postgresql,rabbitmq.SecretName=<rabbitmq_chart_name>-rabbitmq-ha,ingress.enable=true ./reportportal-5.0.0.tgz
   ```
4. To get access to the UI you should create DNS zone and record for created Load Balancer.
    Example: Create k8s.com zone and DNS A record using external LB ip and reportportal.k8s.com like a DNS name