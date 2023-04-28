#### **Assignment 3 Part 1**

Two sensitive data in Django application which needed to be handled in more secure way.

* mysql root password (in db/k/db-deployment.yaml and GiftcardSite/k8/django-deploy.yaml) 
* Secret key (in settings.py) 

For this, I used kubernetes secrets to handle this sensitive data.

To create secrets, below command is used. This created secret my-secrets with two data (mysql_root_password and django_secret_key).

    kubectl create secret generic my-secrets \
    --from-literal=mysql_root_password='thisisatestthing.' \
    --from-literal=django_secret_key='kmgysa#fz+9(z1*=c0ydrjizk*7sthm2ga1z4=^61$cxcq8b$l'\

Now these secrets can be used in yaml files instead of hard coded exposed values.

Changes are implemented in the below files.

**db/k/db-deployment.yaml**

Old code:

    env:
        - name: MYSQL_ROOT_PASSWORD
          value: thisisatestthing.
              
New code:

    env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
                name: my-secrets
                key: mysql_root_password

**GiftcardSite/k8/django-deploy.yaml**

Old code:

    env:
        - name: MYSQL_ROOT_PASSWORD
          value: thisisatestthing.
              
New code:

    env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
                name: my-secrets
                key: mysql_root_password
                
        - name: SECRET_KEY            
              valueFrom:
                secretKeyRef:
                  name: my-secrets     
                  key: django_secret_key



**GiftcardSite/GiftcardSite/settings.py**

Old code:

    SECRET_KEY = 'kmgysa#fz+9(z1*=c0ydrjizk*7sthm2ga1z4=^61$cxcq8b$l'

New Code:

    SECRET_KEY = os.environ.get('SECRET_KEY')

After making these changes, I rebuilt all docker containers. Then I used the command to create kubernetes secrets. I followed the instructions from the assignment to restart the pods.

This is more secure way of implementing secrets and handling sensitive data. To change secret (like changing my sql root password), we just need to run kubectl edit secrets <secret name> command to modify the password. We do not need to go into application and change the code anywhere.

Set of commands which I used to implement secrets:

To remove existing pods:

    kubectl delete -f proxy/k8
    kubectl delete -f GiftcardSite/k8
    kubectl delete -f db/k8
    minikube delete
    
To build dockers and create Pods/Services:  

    minikube start
    eval $(minikube docker-env)
    docker build -t nyuappsec/assign3:v0 .
    docker build -t nyuappsec/assign3-proxy:v0 proxy/
    docker build -t nyuappsec/assign3-db:v0 db/

    kubectl create secret generic my-secrets \
    --from-literal=mysql_root_password='thisisatestthing.' \
    --from-literal=django_secret_key='kmgysa#fz+9(z1*=c0ydrjizk*7sthm2ga1z4=^61$cxcq8b$l'\

    kubectl apply -f db/k8
    kubectl apply -f GiftcardSite/k8
    kubectl apply -f proxy/k8

Verified that the pods and services were created correctly.

    kubectl get pods
    kubectl get service
    
To connect to the site, I used the below command.

    minikube service proxy-service
    
Verified that the website is working as intended even after the change. I was able to register a user, buy/gift/use a gift card, login, logout.
