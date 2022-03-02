# EFS - Stateless/Statefull App - Nginx App with K8S

![alt text](https://github.com/tanersa/kubernetes-jenkins/blob/master/statefulstateless.jpg)

<br />


**ELASTIC FILE SYSTEM (EFS)**

&nbsp; &nbsp; Elastic Block Storage **(EBS)** has a limitation for availability because it's available only in one Availabilty Zone **(AZ).**
EFS is an AWS version of Network File System **(NFS)**. One can consider EFS as managed NFS. 

EFS is almost three times more expensive than EBS. The reason is that EBS is more limited whereas EFS would give you more features to leverage.
The most important benefit of using EFS is that you can share storage **across multiple AZs/multiple ECC2s.** Basically, EFS is managed file system for EC2s.

**To leverage EFS with K8s:**

   -  Create EFS on AWS dashboard. During the creation, choose VPC wherever your EKS Cluster is located. 
   -  Additionally, choose **"Provisioned"** for Throughput mode as we have predictable load. Otherwise, choose **"Bursting"**.
   -  Enable data encryption of data at rest in order to increase security.
   -  Choose all the **AZs** that you would like to store your data.
   -  Add your file policy that you would like to have to use **permissions.** 
 
EFS is created.

Now, we need to install **EFS utility** in order to leverage it.

   -  Go to each instance (worker node) and install EFS utility. Let's say we have 3 instances.
   -  Choose first instance and go to "Connect" and choose "SSH Client" 
   -  Or run below CLI to ssh to instance:

          ssh -i ~/Downloads/eks.pem root@18.116.28.197 "sudo yum install -y amazon-efs-utils"
          
   -  Follow the same steps for second and third instance.
   -  Create a namespace

          kubectl create ns efs
          kubectl get ns
             (efs namespace should be created)
             
   -  Let's create **provisioner.yaml** file for **efs** to talk to **K8S cluster.** We need to provision EFS in our K8S Cluster.
 
          efs-provisioner.yaml
          
          kind: Deployment 
          apiVersion: apps/v1
          metadata:
            name: efs-provisioner
            labels:
              app: efs-provisioner
          spec:
            selector:
              matchLabels:
                app: efs-provisioner
            replicas: 1
            strategy: 
              type: Recreate
            template: 
              metadata: 
                labels:
                  app: efs-provisioner
              spec:
                containers:
                  - name: efs-provisioner
                    image: quay.io/external_storage/efs-provisioner:v0.1.0
                    env: 
                      - name: FILE_SYSTEM_ID
                        value: fs-0eb5ea6c9dfdd1c45
                      - name: AWS_REGION
                        value: us-east-2
                      - name: PROVISIONER_NAME
                        value: efs-sharks/Sharks-EKS-cluster
                    volumeMounts:
                      - name: pv-volume
                        mountPath: /persistentvolumes
                volumes:
                  - name: pv-volume
                    nfs: 
                      server: fs-0eb5ea6c9dfdd1c45.efs.us-east-2.amazonaws.com
                      path: /
                      
   -  Deploy **efs-provisioner.yaml** manifest file

          kubectl apply -f efs.provisioner.yml -n efs
          
          ERROR!
          
Some stuff is missing that is **Permissions.** 

We **do not have** enough permissions to use this deployment file called **efs-provisioner.yaml**

   -  Verify your deployment and pod is up and running

          kubectl get po -n efs
          kubectl get deploy -n efs
                 (All are running)
                 
But we still need to give permissions same as IAM in AWS.

Here, we are going to give our permissions using **create-rbac.yaml** manifest file.

**Permissions** to be given:

   -  Cluster Role
   -  Cluster Role Binding
   -  Service Account

          create-rbac.yaml
          
          apiVersion: rbac.authorization.k8s.io/v1beta1
          kind: ClusterRoleBinding
          metadata:
            name: nfs-provisioner-role-binding
          subjects:
            - kind: ServiceAccount
              name: default
              namespace: efs
          roleRef:
            kind: ClusterRole
            name: cluster-admin
            apiGroup: rbac.authorization.k8s.io

   -  **Deploy** permissions we just defined

          kubectl apply -f create-rbac.yml -n efs
                (ClusterRoleBinding created)
                
   -  Create **Storage Class** from this EFS.

          storage-class.yaml

          kind: StorageClass
          apiVersion: storage.k8s.io/v1
          metadata:
            name: aws-efs
          provisioner: efs-sharks/Sharks-EKS_Cluster
          
          ---
           
         kind: PersistentVolumeClaim 
         apiVersion: v1
         metadata:
           name: efs-wordpress
           annotations:
             volume.beta.kubernetes.io/storage-class: "aws-efs"
         spec:
           accessModes:
             - ReadWriteMany
           resources:
             requests:
               storage: 10Gi
               
          ---
          
         kind: PersistentVolumeClaim 
         apiVersion: v1
         metadata:
           name: efs-mysql
           annotations:
             volume.beta.kubernetes.io/storage-class: "aws-efs"
         spec:
           accessModes:
             - ReadWriteMany
           resources:
             requests:
               storage: 10Gi
               
               
          
               
          
          
    

           
          
          
          
          
          
          
          
          
          
          
          
          
          
          
          
          
          





