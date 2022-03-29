# EFS - Stateless/Statefull App 

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
               
   -  Deploy storage-class.yaml 

               kubectl apply -f storage-class.yaml -n efs
               
                   storage class created
                   persistent volume created
                   persistent volume created
                   
   -  Verify your PVC is visible:

               kubectl get pvc -n efs
                  efs-mysql      PENDING     aws-efs
                  efs-wordpress  PENDING     aws-efs
                  
               kubectl describe pvc efs-mysql -n efs
               
   -  ssh to the instance and run below CLI to see everything running or not.

               sudo ls -a | /var/lib/kubelet/
                    (Everything is running)
                    
**Note**: All PODs, Containers, StorageClasses, and more **ALWAYS** running in **KUBELET.**                    
               
   -  Go inside the pod (worker node) with below CLI.

               kubectl exec -it efs-provisioner-622877cc828 -n efs sh
               
                      It worked!
                      
Lets create **deploy-mysql.yaml** and **deploy-wp.yaml** files

               deploy-mysql.yaml
               
               apiVersion: v1
               kind: Service 
               metadata:
                 name: wordpress-mysql
                 labels:
                   app: wordpress
               spec:
                 ports:
                   - port: 3306
                 selector:
                   app: wordpress
                   tier: mysql
                 clusterIP: None 

               ---
               apiVersion: apps/v1
               kind: Deployment 
               metadata:
                 name: wordpress-mysql
                 labels:
                   app: wordpress
               spec:
                 selector:
                   matchLabels:
                     app: wordpress
                     tier: mysql
                 strategy:
                   type: Recreate
                 template:

                   metadata:
                     labels:
                       app: wordpress
                       tier: mysql
                   spec:
                     containers:
                     - image: mysql:5.6
                       name: mysql
                       env:
                       - name: MYSQL_ROOT_PASSWORD
                         valueFrom:
                           secretKeyRef:
                             name: mysql-pass
                             key: password
                       ports:
                       - containerPort: 3306
                         name: mysql
                       volumeMounts:
                       - name: mysql-persistent-storage
                         mountPath: /var/lib/mysql
                     volumes: 
                     - name: mysql-persistent-storage
                       persistentVolumeClaim:
                         claimName: efs-mysql

              ----------------

               deploy-wp.yaml
               
               apiVersion: v1
               kind: Service 
               metadata:
                 name: wordpress
                 labels:
                   app: wordpress
               spec:
                 ports:
                   - port: 80
                 selector:
                   app: wordpress
                   tier: frontend
                 type: LoadBalancer

               ---
               apiVersion: apps/v1
               kind: Deployment
               metadata:
                 name: wordpress
                 labels:
                   app: wordpress
               spec: 
                 selector:
                   matchLabels:
                     app: wordpress
                     tier: frontend
                 strategy:
                   type: Recreate
                 template:
                   metadata:
                     labels: 
                       app: wordpress
                       tier: frontend
                   spec:
                     containers:
                     - image: wordpress:4.8-apache
                       name: wordpress
                       env:
                       - name: WORDPRESS_DB_HOST
                         value: wordpress-mysql
                       - name: WORDPRESS_DB_PASSWORD
                         valueFrom:
                           secretKeyRef:
                             name: mysql-pass
                             key: password
                       ports:
                       - containerPort: 80
                         name: wordpress
                       volumeMounts:
                       - name: wordpress-persistent-storage
                         mountPath: /var/www/html
                     volumes: 
                     - name: wordpress-persistent-storage
                       persistentVolumeClaim:
                         claimName: efs-wordpress
      
   -  Deploy deploy-mysql.yaml file
  
               kubectl apply -f deploy-mysql.yaml -n efs
               kubectl get pvc -n efs
               kubectl get po -n efs
               kubectl get sc -n efs
               
**Verify if changes whether took place...**

   Go to Load Balancer and check if everything is **healthy**.
   
<br />   
                   
**STATELESS APP**   

&nbsp; &nbsp; K8s is **stateless** by default. Whenever we create a POD, we store data in the POD. This data is available only while POD is running. Once that POD crashes or fails, we would loss data as well. Therefore, PODs use ephemeral storage by default. 

   -  **Deploying Stateless Application**
           
        Create a namespace for my deployment.
           
           kubectl create ns stless
           kubectle get ns
           
        Create manifest yaml file for Deployment.
        
            apiVersion: apps/v1
            kind: Deployment 
            metadata:
              name: redis-master 
            spec:
              selector:
                matchLabels:
                  app: redis 
                  role: master
                  tier: backend
              replicas: 1
              template:
                metadata:
                  labels:
                    app: redis
                    role: master
                    tier: backend
                spec:
                  containers:
                  - name: master
                    image: k8s.gcr.io/redis:e2e
                    resources:
                      requests:
                        cpu: 100m
                        memory: 100Mi
                    ports:
                    - containerPort: 6379 

   -  Note: If replicas value is not specified, there will be one replica of instance by default.

        To deploy to namespace you just created:
        
           kubectl apply -f redis-master.yml -n stless

        To see all PODs, Deployments, and Replicasets in same namespace:
        
           kubectl get all -n stless
           
   -  Note: Whenever you don't specify your namespace, it will be deployed to default namespace. If yaml file is deployed to default namespace, this would create security risk because it would have easy access to others. 

        We created our POD for "redis" master, now we need to create **"Service"**     
        
        Services are like **firewalls** in Kubernetes.
             
        redis-service.yaml
        
            apiVersion: v1
            kind: Service 
            metadata:
              name: redis-master 
              labels:
                app: redis 
                role: master
                tier: backend
            spec:
              ports:
              - port: 6379
                targetPort: 6379
              selector:
                app: redis 
                role: master
                tier: backend


        To deploy to namespace you just created:
        
           kubectl apply -f redis-service.yml -n stless

        To see all PODs, Deployments, and Replicasets in same namespace:
        
           kubectl get svc -n stless    
           
        **DEPLOYMENT KIND:** EC2 INSTANCES
        
        **SERVICE KIIND:** SECURITY GROUP / NETWORK
        
<br />

   Just like creating RDS, we create Master and Stand By DBs. Therefore, we create slave dbs now.
   
        redis-slave.yml
        
         apiVersion: apps/v1
         kind: Deployment 
         metadata: 
           name: redis-slave
         spec:
           selector:
             matchLabels:
               app: redis 
               role: slave
               tier: backend
           replicas: 2
           template:
             metadata:
               labels:
                 app: redis 
                 role: slave
                 tier: backend
             spec:
               containers:
               - name: slave
                 image: gcr.io/google_samples/gb-redisslave:v1
                 resources:
                   requests:
                     cpu: 100m
                     memory: 100Mi
                 env:
                 - name: GET_HOSTS_FROM
                   value: dns
                 ports:
                 - containerPort: 6379
                 
      
   Deploy redis-slave yaml file to stless namespace
        
           kubectl apply -f redis.yml -n stless
           
   To see all DBs in stless app:
   
           kubectl get po -n stless
           
              redis-master
              redis-slave
              redis-slave
              
   Now, lets create a **"Service"** for redis-slave for having security group/network
   
           redis-slave-service.yaml
           
            apiVersion: v1
            kind: Service 
            metadata:
              name: redis-slave
              labels:
                app: redis 
                role: slave
                tier: backend
            spec:
              ports:
              - port: 6379
              selector:
                app: redis 
                role: slave
                tier: backend
                
   To deploy service...
   
            kubectl apply -f redis-slave-service.yaml -n stless
            
   To verify service...
   
            kubectl get svc -o wide -n stless
            
   Now, we need to create **front-end application** and deploy.

            redis-frontend.yaml
            
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: frontend
            spec:
              selector:
                matchLabels:
                  app: guestbook
                  tier: frontend
              replicas: 3
              template:
                metadata:
                  labels:
                    app: guestbook
                    tier: frontend
                spec:
                  containers:
                  - name: php-redis
                    image: gcr.io/google_samples/gb-frontend:v4
                    resources:
                      requests:
                        memory: 100Mi
                        cpu: 100m
                    env:
                    - name: GET_HOSTS_FROM
                      value: dns 
                    ports:
                    - containerPort: 80
                    
   Deploy front-end app 
   
            kubectl apply -f redis-frontend.yaml -n stless
            
   Next, deploy front-end-service app
   
            redis-frontend-service.yaml
            
            apiVersion: v1
            kind: Service 
            metadata:
              name: frontend
              labels:
                app: guestbook
                tier: frontend
            spec:
              type: LoadBalancer
              ports:
              - port: 80
              selector:
                app: guestbook
                tier: frontend
                
To deploy front-end app and service:
         
          kubectl apply -f redis-frontend.yaml -n stless
          kubectl apply -f redis-frontend-service.yaml -n stless
          
To verify stateless app. go to AWS and Load Balancer:
 
         Verify Load Balancer is created
         
         Check Instances
         
         Health Check
         
         Listeners
         
         Go to description and copy DNS name and paste it to your browser.
         
**Now, we deployed our first Stateless Application**  

<br />

**TO JUSTIFY THIS IS AN STATELESS APPLICATION:**

Lets delete one of the  PODs...

         kubectl delete po front-end-3434324323432-66
         
Go to browser and refresh the page...

Application should be still up and running

What happens is that **K8s Self Healing Mechanism** works perfect.

   **Once worker node goes down, it automatically cordons so it deploys the pod to healthy worker node.**
   
However, data is gone because our application is **stateless**. We deleted worker node and once it's deleted, PODs get deleted as well. Therefore, we loose the data. 

<br />

**STATEFULL APPLICATION**

&nbsp; &nbsp; Whenever it comes to statefull application, we are going to use persistent volumes (storages).

For this purpose, AWS has **Elastic Block Storage (EBS)**

   -  Note: Whenever we create EKS Cluster, it comes with a storage class. Default storage class is **gp2** in K8s.

K8s uses thiss default storage classs to store Kubernetes related **configurations.** However, we do not use this default storage class to store data for our application. 

   Lets create our first yaml file for storage class...
   
         storageclass.yaml
         
         kind: StorageClass
         apiVersion: storage.k8s.io/v1
         metadata:
           name: statefull
         provisioner: kubernetes.io/aws-ebs
         parameters:
           type: gp2
         reclaimPolicy: Retain 
         mountOptions: 
           - debug 
           
   -  Note: As we give **Retain** value for  reclaimPolicy, we stated that even our POD goes down, we would like to keep this storage class so that will be **Persistent Storage** for us.    

   Create new namespace for our **statefull storage class:**
   
         kubectl create ns sfull
         
   Deploy statefull application
   
         kubectl apply -f storageclass.yaml -n sfull
         
   Verify storage class creation
   
         kubectl get sc -n sfull
         
   We just created storage class but we did not create **persistent volume** yet. Now, we have to create persistent volumes from this storage class. 
   
   Currently, our default storage class is gp2 but we need to **patch** our own storage class as default one in order to create persistent volume in our custom storage class.   
   
   To patch sfull storage class as default storage class:
   
         kubectl patch storageclass statefull -p '{"metada":{"annotation":{"storageclass.kubernetes.io/is-default-class":"true"}}} -n sfull
         
Let's create now our persistent volumes:

      pvc.yaml
      
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: mysql-pv-claim
        labels:
          app: wordpress
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
      --- 
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: wp-pv-claim
        labels:
          app: wordpress
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
            
   -  Note: We created two Persistent Volume Claims (PVC). One is for BackEnd and other one is for FrontEnd. 

   We had accessModes as **ReadWriteOnce** because its EBS and EBS has limitation for only one **Availability Zone (AZ)**. 
   
   Deploy your persistent volume manifest yaml file as follows.
   
            kubectl apply -f pvc.yaml -n sfull
            
   
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         



            
   
                
          
           
           


        
        
           
           

   


               
               

                             
               
               

              
                  
                  
                  
                  
          
               
          
          
    

           
          
          
          
          
          
          
          
          
          
          
          
          
          
          
          
          
          





