# Kubernetes Practice Questions

---

## Q. 1

### Task

In certain cases, applications deployed on the Kubernetes cluster require specific configurations or setup changes before launching the app container. The Nautilus DevOps team has devised a solution using init containers to fulfill these prerequisites during deployment. Below is an initial test scenario:

a. Create a Pod named **red-xfusion-t1q5**. It must have an init container named **red-init-xfusion-t1q5**, it should utilise image `debian` with the latest tag. The command `/bin/bash`, `-c` should be used with arguments `echo "Welcome!"`

b. The main container name should be **red-main-xfusion-t1q5** and it should utilise image `debian` with the latest tag. The Command: `/bin/bash`, `-c` should be used with arguments `sleep 1000`

This scenario demonstrates the use of init containers to fulfil pre-requisites before deploying the main application container in the Kubernetes Pod.

> **Note:** The kubectl utility on jump-host has been configured to work with the kubernetes cluster.

---

## Q. 2

### Task

The Nautilus DevOps team is actively engaged in practicing pods and services deployment on the Kubernetes platform, preparing for the migration of numerous applications to this environment. Recently, a team member has been assigned the task of creating a pod with specific requirements:

a. Create a pod named **pod-httpd-t1q1** utilizing the `httpd` image, ensuring the use of the latest tag, denoted as `httpd:latest` (remember to mention tag name while defining the image).

b. Set the app label to `httpd_app_t1q1`, and name the container as **httpd-container-t1q1**.

This task aims to establish a new pod adhering to defined image specifications and container naming conventions.

> **Note:** The kubectl utility on jump-host has been configured to work with the kubernetes cluster.

---

## Q. 3

### Task

The Nautilus DevOps team has started practicing some pods and services deployment on Kubernetes platform, as they are planning to migrate most of their applications on Kubernetes. Recently one of the team members has been assigned a task to create a deployment as per details mentioned below:

Create a deployment named **httpd-t2q1** to deploy the application `httpd-t2q1` using the image `httpd:latest` (remember to mention the tag as well).

> **Note:** The kubectl utility on jump-host has been configured to work with the kubernetes cluster.

---

## Q. 4

### Task

This morning the Nautilus DevOps team rolled out a new release for one of the applications. Recently one of the customers logged a complaint which seems to be about a bug related to the recent release. Therefore, the team wants to rollback the recent release.

There is a deployment named **nginx-deployment-t2q2**, roll it back to the previous revision.

> **Note:** The kubectl utility on jump-host has been configured to work with the kubernetes cluster.

---

## Q. 5

### Task

The Nautilus DevOps team plans to deploy applications on a Kubernetes cluster for the migration of some existing applications. Recently, a team member has been tasked with creating below components:

a. Create a ReplicaSet named **httpd-replicaset-t3q4** using `httpd` image with latest tag only (remember to mention tag i.e `httpd:latest`).

b. Label `app` should be `httpd_app_t3q4`, label `type` should be `front-end-t3q4`.

c. The container should be named as **httpd-container-t3q4**, also make sure replicas counts are **4**.

> **Note:** The kubectl utility on jump-host has been configured to work with the kubernetes cluster.

---

## Q. 6

### Task

The Nautilus DevOps team aims to establish a ReplicaSet to deploy multiple pods for hosting applications requiring a highly available infrastructure. Here are the specific details to create the ReplicaSet:

a. Create a ReplicaSet using the `httpd` image with the latest tag.

b. Name the ReplicaSet **httpd-replicaset-t3q5**.

c. Ensure all pods are in the **Running** state after deployment.

d. The ReplicaSet should manage **3** replicas (adjustable based on your needs).

e. The pods managed by the ReplicaSet should have the label `app: httpd-t3q5`.

> **Note:** The kubectl utility on jump-host has been configured to work with the Kubernetes cluster.

---

## Q. 7

### Task

Last week, our team successfully deployed a Nginx and PHP-FPM based setup on the Kubernetes cluster, functioning seamlessly. However, this morning, a team member made changes that resulted in disruptions, rendering the setup inoperable.

The pod named **nginx-phpfpm-t4q3** and the associated config map named **nginx-config-t4q3** are experiencing issues following recent modifications. Your task is to diagnose the issue and restore functionality to the setup.

Once the issue is resolved, copy the `/home/thor/index.php` file from the jump-host to the `nginx-container`, placing it within the document root `/var/www/html`. Subsequently, ensure the website becomes accessible via the **Website** button located in the top bar.

Please proceed with troubleshooting the identified issues and implementing the necessary fixes. Afterward, complete the file transfer and verification of website accessibility. Also, feel free to re-create the pod if needed.

> **Note:** The kubectl utility on jump-host has been configured to work with the kubernetes cluster.

---

## Q. 8

### Task

Despite diligent efforts, the DevOps team encountered difficulties deploying the **orange-app-deployment-t4q6** on the Kubernetes cluster. Regrettably, this app is currently inaccessible. Your prompt attention to resolving this issue is crucial.

The **orange-app-deployment-t4q6** is currently facing accessibility issues on the Kubernetes cluster. Your immediate objective is to investigate and rectify this issue to restore the app's functionality and accessibility.

Please prioritize diagnosing the root cause of the deployment issue and apply necessary fixes to ensure the successful deployment and accessibility of the **orange-app-deployment-t4q6**.

> **Note:** The kubectl on jump-host has been configured to work with the kubernetes cluster.

---

## Q. 9

### Task

During an investigating it was found that one of the applications on the Kubernetes cluster is having some issues, the team discovered that the service was configured with an incorrect target port. We need to update the service as follows:

Update **service-t5q4** service to use target port **80**.

---

## Q. 10

### Task

An application was deployed on the Kubernetes cluster, the deployment name is **deployment-t5q5**. There is a service named **service-t5q5** associated with this deployment. We need to make below changes in this service.

Update **service-t5q5** service to another match label `component: front-end-t5q5`.
