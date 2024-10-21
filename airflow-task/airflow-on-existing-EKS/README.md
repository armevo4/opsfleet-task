## Deploy Apache Airflow on AWS Elastic Kubernetes Service (EKS)

### Deploy Airflow on AWS EKS
Let us install Apache Airflow in the EKS cluster using the helm chart.

1. Create a new namespace.

```
    kubectl create namespace airflow
```

2. Add the Helm chart repository.

```

    helm repo add apache-airflow https://airflow.apache.org
    
```

3. Update your Helm repository.

```

    helm repo update
    
```

4. Deploy Airflow using the remote Helm Chart

```

helm install airflow apache-airflow/airflow --namespace airflow   --debug
    
```

You will get the Airflow webserver and default Postgres connection credentials in the output. Copy them and save them somewhere.

5. Examine the deployments by getting the Pods

```

    Kubectl get pods -n airflow
    
```

The Airflow instance is set up in EKS. All the airflow pods should be running.

### Let’s prepare Airflow to run our first DAG
At this point, Airflow is deployed using the default configuration. Let's see how we can get the default values from the helm chart on our local machine, modify it, and update a new release.

1. Save the configuration values from the helm chart by running the below command.

```

    helm show values apache-airflow/airflow > values.yaml
    
```

This command generates a file named

```

    values.yaml
    
```

in your current directory, which you can modify and save as needed.

2. Check the release version of the helm chart by running the following command.

```

    helm ls -n airflow
    
```

3. Let us add the ingress configuration to access the airflow instance over the internet. 

We need to deploy an ingress controller in the EKS cluster first. The commands below will install the NGINX ingress controller from the helm repository. 

```

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginxhelm install nginx-ingress ingress-nginx/ingress-nginx --namespace airflow-ingress --create-namespace --set controller.replicaCount=2kubectl get pods -n airflow-ingress
    
```

Note - All the pods should be running.

```

    kubectl get service nginx-ingress-controller --namespace airflow-ingress
    
```

Look for the external IP in the output of the get service command.

After installing the ingress controller, add the required configuration in the values.yaml file and save the file. There is a section dedicated to the ingress configuration.

```

# Ingress configuration
ingress:
  enabled: true
  web:
    enabled: true
    annotations: {}
    path: "/"
    pathType: "ImplementationSpecific"
    host: 
    ingressClassName: "nginx"
    
```

After the changes to the values in the values.yaml file, we run the helm upgrade command to deploy the changes and create a new release version.

By default, the Helm Chart deploys its own Postgres instance, but using a managed Postgres instance is recommended instead.

You can modify the Helm Chart’s values.yaml file to add  configuration of the managed database and volumes

```

metadataConnection:
             user: postgres
             pass: postgres
             protocol: postgresql
             host: 
             port: 5432
             db: postgres
             sslmode: disable
    
```

Run the helm upgrade command to implement the changes done above.

```

    helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
    
```

Check the release version after the above command is run successfully. You should observe that the revision has changed to 2.


### Accessing Airflow UI

We will use port-forwarding to access the Airflow UI in this tutorial. Run the below command and access “localhost:8080” on the browser.

```

    helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
    
```

Use the default webserver credentials saved in the above section, “Installing Airflow Helm chart.”


![alt text](images-for-readme/README.avif)
![alt text](images-for-readme/pic2.avif)

### Create your first Airflow DAG (in Git)
No DAGs have been added to our Airflow deployment yet. Let us see how we can add them.

To Set up a private GitHub repository for DAG, you can create a new one using the Github website's UI.

![alt text](images-for-readme/pic3.avif)

You can also install the git command line interface on your local machine and run commands to initialize an empty git repo.

```
    git init
```

### Adding DAG configs to the git repo

Once the git repo is initialized, create a DAG file like “sample_dag.py” and push it to the remote branch.

```

git add .
git commit -m 'Adding first DAG'
git remote add origin
git push -u origin main
    
```

Integrate Airflow with a private Git repo

To integrate Airflow with a private Git repository, you will need credentials, i.e. username /password or an SSH key.

We will use the SSH key to connect to the git repo. Skip the first step below if the SSH Key already exists in your Github account.

1. [Skip if it already exists] Generate an SSH key in your local machine and add it to the GitHub account (If not already present). 

```

ssh-keygen -t ed25519 -C ""
    
```

2. Create a generic secret in the namespace where airflow is deployed. This secret contains your SSH key.

```

kubectl create secret generic airflow-ssh-git-secret --from-file=gitSshKey= -n airflow
    
```

3. Update the Git configuration in values.yaml file and run helm update command like in the above section.

```

gitSync:
    enabled: true
    repo: 
    branch: 
    rev: HEAD
    depth: 1
    maxFailures: 0
    subPath: ""
sshKeySecret: airflow-ssh-git-secret
    
```

Below is a “sample_dag.py” that demonstrates a simple workflow. 

```

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2024, 8, 8),
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}
dag = DAG('hello_world', default_args=default_args, schedule_interval=timedelta(days=1))
t1 = BashOperator(
    task_id='say_hello',
    bash_command='echo "Hello World from Airflow!"',
    dag=dag,
)
    
```

Upon completion, you can see the DAGs in the UI interface. Airflow automatically detects new DAGs, but you can manually refresh the DAGs list in the Airflow UI by clicking the "Refresh" button on the DAGs page.

![alt text](images-for-readme/pic4.avif)

![alt text](images-for-readme/pic5.avif)

![alt text](images-for-readme/pic6.avif)

The UI has many options/settings to experiment with, such as code, graphs, audit logs, etc.

You can also check the EKS cluster’s activity and DAG dashboard from the Activity tab.

![alt text](images-for-readme/pic7.avif)

![alt text](images-for-readme/pic8.avif)


### Run the Airflow job
DAGs can be scheduled to run or triggered manually from the UI interface. There is a run button on the rightmost side of the DAG table.

![alt text](images-for-readme/pic9.avif)

Also, it can be triggered from within the DAG.

![alt text](images-for-readme/pic10.avif)

Make your Airflow on Kubernetes Production-Grade
Apache Airflow is a powerful tool for orchestrating workflows, but making it production-ready requires careful attention to several key areas. Below, we explore strategies to enhance security, performance, monitoring, and ensure high availability in your Airflow deployment.

‍

### 1. Improved Security
a. Role-Based Access Control (RBAC)
. Implementation: Enable RBAC in Airflow to ensure only authorized users can access specific features and data.

. Benefits: Limits access to critical areas and reduces the risk of unauthorized changes or data breaches.

b. Secrets Management
. Implementation: Integrate with external secret management tools like AWS Secrets Manager, HashiCorp Vault, or Kubernetes secrets.

. Benefits: Securely store sensitive information like API keys and database passwords, keeping them out of your codebase.


c. Network Security

. Implementation: Use network policies and security groups to restrict Airflow's web interface and API access.

. Benefits: Minimizes exposure to potential attacks by limiting network access to trusted sources only.
‍

### 2. Improved Performance
a. Optimized Resource Allocation

. Implementation: Right-size your Kubernetes pods and nodes based on the workload demand. Use Kubernetes Horizontal Pod Autoscaler (HPA) to scale Airflow resources dynamically and cluster autoscaler to scale nodes.

. Benefits: Ensures efficient use of resources, reduces costs, and prevents bottlenecks during peak loads.

b. Task Parallelism

. Implementation: Configure Airflow to handle parallel task execution by optimizing the number of worker pods and setting appropriate concurrency limits.

. Benefits: Accelerates workflow execution by running multiple tasks simultaneously, improving overall performance.


c. Use of ARM Instances

. Implementation: Consider running workloads on ARM-based instances like AWS Graviton for cost efficiency.

. Benefits: ARM instances often provide a better cost-to-performance ratio, especially for compute-intensive tasks.

d. Use of HTTPS for ingress host

. Implementation: Consider having HTTPS for the Airflow URL using TLS/SSL certificates with the Ingress controller in Kubernetes.

. Benefits: HTTPS encrypts data to enhance the security of information being transferred. This is especially crucial when handling sensitive data, as encryption helps protect it from unauthorized access during transmission.


 

### 3. Monitoring
a. Metrics Collection and Alerting

. Implementation: Integrate Airflow with Prometheus to collect metrics on task performance, resource usage, and system health. Tools like Grafana or Prometheus Alertmanagercan can set up alerts based on critical metrics and log events.

. Benefits: It provides visibility into Airflow’s performance, allowing you to identify and address issues proactively and enabling quick response to potential problems, reducing downtime and maintaining workflow reliability.

b. Logs Collection

. Implementation: Set up centralized logging with tools like Elasticsearch, Logstash, Kibana (ELK stack or EFK stack), or Grafana Loki.

. Benefits: Simplifies troubleshooting by consolidating logs from all Airflow components into a single, searchable interface.


### ‍4. High Availability
#### a. Redundant Components

. Implementation: Deploy multiple replicas of Airflow’s web server, scheduler, and worker nodes to ensure redundancy.

. Benefits: Increases resilience by preventing single points of failure, ensuring that workflows continue even if one component goes down.

To deploy multiple pods in Apache Airflow using a Helm chart, follow these steps:

1. Set Replicas for the Scheduler:

In your values.yaml file set the scheduler.replicas to the desired number of replicas. For example:

```

scheduler:
  replicas: 2
    
```

2. Set Replicas for the Web Server:

Similarly, set the web.replicas to deploy multiple web server pods:

```

web:
  replicas: 2
    
```

3. Deploy the Helm Chart:

Apply the Helm chart with the updated values.yaml file:

```

helm upgrade --install airflow apache-airflow/airflow -f values.yaml
    
```

This configuration ensures that multiple scheduler and web server pods are deployed, contributing to the high availability of your Airflow setup.



#### b. Database High Availability
. Implementation: Use a highly available database solution like Amazon RDS with Multi-AZ deployment for Airflow’s metadata database.

. Benefits: Ensures continuous operation and data integrity even during a database failure.

#### c. Backup and Disaster Recovery
. Implementation: Regularly backup Airflow’s database and configuration files. Implement a disaster recovery plan that includes rapid failover procedures.

. Benefits: Protects against data loss and enables quick recovery in case of catastrophic failures.

