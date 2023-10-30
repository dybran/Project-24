## __BUILDING ELASTIC KUBERNETES SERVICE (EKS) WITH TERRAFORM__

In this extensive project, our primary emphasis will be on the practical real-world implementation of Kubernetes deployment and management. This involves a series of tasks geared toward guaranteeing the resilience and scalability of our system. The key objectives include:

- I plan to employ Terraform to establish a Kubernetes EKS cluster and dynamically integrate adaptable worker nodes into the system. 

- Furthermore, I intend to deploy multiple applications using __HELM__, leveraging Kubernetes objects in conjunction with Helm. This approach includes dynamic provisioning of volumes to enable stateful pods.

- Finally, I will execute the deployment process via CI/CD using Jenkins.

__Building EKS with Terraform__

Create a directory on your local machine - __eks__ and create an s3 bucket

`$ mkdir eks && cd eks`

`$ aws s3api create-bucket --bucket eks-terraform-deploy --region us-east-1`

![](./images/ekst.PNG)
![](./images/ekstt.PNG)







