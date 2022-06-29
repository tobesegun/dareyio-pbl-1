# PERSISTING DATA IN KUBERNETES
Create EKS cluster first before the below section
- Kubernetes supports many types of volumes. A Pod can use any number of volume types simultaneously. Ephemeral volume types have a lifetime of a pod, but persistent volumes exist beyond the lifetime of a pod. When a pod ceases to exist, Kubernetes destroys ephemeral volumes; however, Kubernetes does not destroy persistent volumes. For any kind of volume in a given pod, data is preserved across container restarts.

## awsElasticBlockStore
An awsElasticBlockStore volume mounts an Amazon Web Services (AWS) EBS volume into your pod. The contents of an EBS volume are persisted and the volume is only unmmounted when the pod crashes, or terminates. This means that an EBS volume can be pre-populated with data, and that data can be shared between pods.

## Step1
Lets see what it looks like for our Nginx pod to persist data using awsElasticBlockStore volume

```
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "<volume id>"
          fsType: ext4
EOF
```
the volume section indicates the type of volume to be used to ensure persistence.

If you notice the config above carefully, you will realise that there is need to provide a volumeID before the deployment will work. Therefore, You must create an EBS volume by using aws ec2 create-volume command or the AWS console.

Before you create a volume, lets run the nginx deployment into kubernetes without a volume.

```
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
```
## Step2
Verify that the pod is running
Check the logs of the pod
Exec into the pod and navigate to the nginx configuration file /etc/nginx/conf.d
Open the config files to see the default configuration

## Step3
Now that we have the pod running without a volume, Lets now create a volume from the AWS console.

In your AWS console, head over to the EC2 section and scroll down to the Elastic Block Storage menu.
Click on Volumes
At the top right, click on Create Volume 
The create volume selection should be like
Copy the VolumeID
<img width="1800" alt="Screenshot 2022-06-29 at 23 28 47" src="https://user-images.githubusercontent.com/33035619/176560993-9a9bb629-99ca-480f-b857-29aa51606a57.png">

<img width="1800" alt="Screenshot 2022-06-29 at 23 30 12" src="https://user-images.githubusercontent.com/33035619/176561157-66a0660e-f965-4062-b48f-5fc379c104c1.png">


## Step4
Update the deployment configuration with the volume spec.

```
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "vol-0e194e56f1b5302ee"
          fsType: ext4
EOF
```
## step5
At this point, even though the pod can be used for a stateful application, the configuration is not yet complete. This is because, the volume is not yet mounted onto any specific filesystem inside the container. The directory /usr/share/nginx/html which holds the software/website code is still ephemeral, and if there is any kind of update to the index.html file, the new changes will only be there for as long as the pod is still running. If the pod dies after, all previously written data will be erased.

To complete the configuration, we will need to add another section to the deployment yaml manifest. The volumeMounts which basically answers the question "Where should this Volume be mounted inside the container?" Mounting a volume to a directory means that all data written to the directory will be stored on that volume.

Lets do that now.

```
cat <<EOF | tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "  vol-07b537651bbe68be0"
          fsType: ext4
EOF
```
Notice the newly added section:

```
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/
```


<img width="520" alt="Screenshot 2022-06-29 at 23 25 40" src="https://user-images.githubusercontent.com/33035619/176561270-7b4bafca-546f-4f80-bf57-85c6e91d1e83.png">


- A more elegant way to achieve this is through Persistent Volume and Persistent Volume claims.

In kubernetes, there are many elegant ways of persisting data. Each of which is used to satisfy different use cases. Lets take a look at the different options available.

Persistent Volume (PV) and Persistent Volume Claim (PVC)
configMap

#MANAGING VOLUMES DYNAMICALLY WITH PVS AND PVCS
Kubernetes provides API objects for storage management such that, the lower level details of volume provisioning, storage allocation, access management etc are all abstracted away from the user, and all you have to do is present manifest files that describes what you want to get done.

PVs are volume plugins that have a lifecycle completely independent of any individual Pod that uses the PV. This means that even when a pod dies, the PV remains. A PV is a piece of storage in the cluster that is either provisioned by an administrator through a manifest file, or it can be dynamically created if a storage class has been pre-configured.

Creating a PV manually is like what we have done previously where with creating the volume from the console. As much as possible, we should allow PVs to be created automatically just be adding it to the container spec iin deployments. But without a storageclass present in the cluster, PVs cannot be automatically created.

Run the command below to check if you already have a storageclass in your cluster kubectl get storageclass
kubectl get storageclass

```
  NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
  gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  18d
  
```

If there is no storage class in your cluster, below manifest is an example of how one would be created
```
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: gp2
    annotations:
      storageclass.kubernetes.io/is-default-class: "true"
  provisioner: kubernetes.io/aws-ebs
  parameters:
    type: gp2
    fsType: ext4 
```



## step6 Now lets create some persistence for our nginx deployment. We will use 2 different approaches.

Create a manifest file for a PVC, and based on the gp2 storageClass a PV will be dynamically created
```
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nginx-volume-claim
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi
      storageClassName: gp2
  ```
Apply the manifest file and you will get an output like below
persistentvolumeclaim/nginx-volume-claim created

Run get on the pvc and you will notice that it is in pending state. 
kubectl get pvc
NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE
nginx-volume-claim Pending gp2 61s

<img width="370" alt="Screenshot 2022-06-30 at 00 07 24" src="https://user-images.githubusercontent.com/33035619/176561332-66bf8a6b-ede9-4236-815a-67124bb4a11e.png">
<img width="654" alt="Screenshot 2022-06-30 at 00 07 34" src="https://user-images.githubusercontent.com/33035619/176561337-849dbf64-3cc7-4b9c-bbe9-f9d5696074fd.png">

# CONFIGMAP
- Using configMaps for persistence is not something you would consider for data storage. Rather it is a way to manage configuration files and ensure they are not lost as a result of Pod replacement.

to demonstrate this, we will use the HTML file that came with Nginx. This file can be found in /usr/share/nginx/html/index.html  directory.

- Lets go through the below process so that you can see an example of a configMap use case.

- Remove the volumeMounts and PVC sections of the manifest and use kubectl to apply the configuration

- port forward the service and ensure that you are able to see the "Welcome to nginx" page

- exec into the running container and keep a copy of the index.html file somewhere. For example

```
kubectl exec -it nginx-deployment-79d8c764bc-j6sp9 -- bash
  cat /usr/share/nginx/html/index.html 
  ```

## Persisting configuration data with configMaps
According to the official documentation of configMaps, A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

In our own use case here, We will use configMap to create a file in a volume.

The manifest file we look like:

```
cat <<EOF | tee ./nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: website-index-file
data:
  # file to be mounted inside a volume
  index-file: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
EOF
```
- Apply the new manifest file
```
 kubectl apply -f nginx-configmap.yaml 
 ```
 
 <img width="612" alt="Screenshot 2022-06-30 at 00 41 35" src="https://user-images.githubusercontent.com/33035619/176563956-df819082-f557-4ca5-861d-e4e16fae9f8d.png">
 
 - Update the deployment file to use the configmap in the volumeMounts section
```
cat <<EOF | tee ./nginx-pod-with-cm.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
          - name: config
            mountPath: /usr/share/nginx/html
            readOnly: true
      volumes:
      - name: config
        configMap:
          name: website-index-file
          items:
          - key: index-file
            path: index.html
EOF
```
- Now the index.html file is no longer ephemeral because it is using a configMap that has been mounted onto the filesystem. This is now evident when you exec into the pod and list the /usr/share/nginx/html directory

```
  root@nginx-deployment-84b799b888-fqzwk:/# ls -ltr  /usr/share/nginx/html
  lrwxrwxrwx 1 root root 17 Feb 19 16:16 index.html -> ..data/index.html
  ```

You can now see that the index.html is now a soft link to ../data

- Accessing the site will not change anything at this time because the same html file is being loaded through configmap.

- But if you make any change to the content of the html file through the configmap, and restart the pod, all your changes will persist.

Lets try that;

- List the available configmaps. You can either use kubectl get configmap or kubectl get cm

```
kubectl get cm
NAME                 DATA   AGE
kube-root-ca.crt     1      17d
website-index-file   1      46m
```
- We are interested in the website-index-file configmap

- Update the configmap. You can either update the manifest file, or the kubernetes object directly. Lets use the latter approach this time.

```
kubectl edit cm website-index-file 
```


It will open up a vim editor, or whatever default editor your system is configured to use. Update the content as you like. "Only the html data section", then save the file.

You should see an output like this


configmap/website-index-file edited

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: website-index-file
data:
```
  ## file to be mounted inside a volume
  ```
  index-file: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to DAREY.IO!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to DAREY.IO!</h1>
    <p>If you see this page, It means you have successfully updated the configMap data in Kubernetes.</p>

    <p>For online documentation and support please refer to
    <a href="http://DAREY.IO/">DAREY.IO</a>.<br/>
    Commercial support is available at
    <a href="http://DAREY.IO/">DAREY.IO</a>.</p>

    <p><em>Thank you and make sure you are on Darey's Masterclass Program.</em></p>
    </body>
    </html>
  ```
    
## Without restarting the pod, your site should be loaded automatically.
If you wish to restart the deployment for any reason, simply use the command

```
   kubectl rollout restart deploy nginx-deployment 

   ```
   
   
output:

deployment.apps/nginx-deployment restarted


<img width="1057" alt="Screenshot 2022-06-30 at 00 30 05" src="https://user-images.githubusercontent.com/33035619/176562741-860500e9-7741-43e8-846e-78193efc2983.png">


