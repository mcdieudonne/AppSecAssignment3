#### **Assignment 3 Part 2**

Currently db/Dockerfile modifies the default MYSQL docker image to include a sql script that both performs migrations and seeds the database at the same time. This has a limitation that in order to perform changes to the database, the MYSQL pod must be destroyed and a new one must go up in its place. Instead, we want to be able to run migrations whenever there is a change in the models, and only seed the data once when the MYSQL Database is created.

**Solution:**

Modify db/Dockerfile and remove the code which is using SQL script to create tables and seed the database.

Old dockerfile:

    FROM mysql:latest
    
    RUN mkdir /data
    COPY ./products.csv /data/products.csv
    COPY ./users.csv /data/users.csv
    COPY ./setup.sql /docker-entrypoint-initdb.d/setup.sql
    
    ENTRYPOINT ["/entrypoint.sh"]
    CMD ["mysqld", "--secure-file-priv=/data"]
    #CMD ["--secure-file-priv=/"]
    
New dockerfile:

    FROM mysql:latest
    
    ENTRYPOINT ["/entrypoint.sh"]
    CMD ["mysqld", "--secure-file-priv=/data"]
    #CMD ["--secure-file-priv=/"]

This change will make sure that migration and seeding does not happen via SQL script when my sql container is created.

For now I did not delete files setup.sql, products.csv and users.csv from the repository but we do not need them.

#### **Migration**

Created a kubernetes job django-migrations.yaml to run django migration. File is stored at new directory k8MigrateSeed and not in K8 as we do not want to run migration when we create Django pod. Instead we will run them manually after all three pods (my Sql, Django and Proxy) are up and running. 

This kubernetes job will run the below commands (please check GiftcardSite/k8MigrateSeed/django-migrations.yaml for full job code).

    python3 manage.py makemigrations LegacySite
    python3 manage.py migrate

 makemigrations is responsible for creating new migrations based on the changes made to models.
 
 migrate is responsible for applying and unapplying migrations.
 
 When this job is complete, it will result in creating or altering tables in database with respect to models or change in models. 
 
 Steps: (assuming that we start from scratch)
 
    minikube start
    eval $(minikube docker-env)
    docker build -t nyuappsec/assign3:v0 .
    docker build -t nyuappsec/assign3-proxy:v0 proxy/
    docker build -t nyuappsec/assign3-db:v0 db/
    
    kubectl create secret generic my-secrets \
    --from-literal=mysql_root_password='thisisatestthing.' \
    --from-literal=django_secret_key='kmgysa#fz+9(z1*=c0ydrjizk*7sthm2ga1z4=^61$cxcq8b$l'
    
    kubectl apply -f db/k8
    kubectl apply -f GiftcardSite/k8
    kubectl apply -f proxy/k8
 
 Use kubectl get pods to make sure all PODs are up and running.
 
```Bash
appsecstudent@appsecstudent-VirtualBox:~/AppSecAssignment3/AppSecAssignment3$ kubectl get pods
NAME                                         READY   STATUS    RESTARTS   AGE
assignment3-django-deploy-7cb68dc4cf-l2gnp   1/1     Running   0          53s
mysql-container-79c5db5bdf-kz8ls             1/1     Running   0          63s
proxy-84787764d7-xcj9t                       1/1     Running   0          44s

```

Run the below command to run migration.

    kubectl apply -f GiftcardSite/k8MigrateSeed/django-migrations.yaml
    
Check the status and wait for its completion.

```Bash

appsecstudent@appsecstudent-VirtualBox:~/AppSecAssignment3/AppSecAssignment3$ kubectl apply -f GiftcardSite/k8MigrateSeed/django-migrations.yaml
job.batch/assignment3-django-migrations created


appsecstudent@appsecstudent-VirtualBox:~/AppSecAssignment3/AppSecAssignment3$ kubectl get pods
NAME                                         READY   STATUS      RESTARTS   AGE
assignment3-django-deploy-7cb68dc4cf-l2gnp   1/1     Running     0          91s
assignment3-django-migrations-snqzd          0/1     Completed   0          7s
mysql-container-79c5db5bdf-kz8ls             1/1     Running     0          101s
proxy-84787764d7-xcj9t                       1/1     Running     0          82s
```

If the status of assignment3-django-migrations job is completed, then it means it was successful. To verify that tables are created, get a shell inside the mysql pod's container and run ms sql queries.

```Bash
appsecstudent@appsecstudent-VirtualBox:~/AppSecAssignment3/AppSecAssignment3$ kubectl exec --stdin --tty mysql-container-79c5db5bdf-kz8ls  -- bash
bash-4.4# mysql -uroot -pthisisatestthing. -DGiftcardSiteDB -e "describe LegacySite_user"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+-------------+------+-----+---------+----------------+
| Field      | Type        | Null | Key | Default | Extra          |
+------------+-------------+------+-----+---------+----------------+
| id         | int         | NO   | PRI | NULL    | auto_increment |
| last_login | datetime(6) | YES  |     | NULL    |                |
| username   | varchar(30) | NO   | UNI | NULL    |                |
| password   | varchar(97) | NO   |     | NULL    |                |
+------------+-------------+------+-----+---------+----------------+
bash-4.4# mysql -uroot -pthisisatestthing. -DGiftcardSiteDB -e "describe LegacySite_product"
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+--------------+------+-----+---------+----------------+
| Field              | Type         | Null | Key | Default | Extra          |
+--------------------+--------------+------+-----+---------+----------------+
| product_id         | int          | NO   | PRI | NULL    | auto_increment |
| product_name       | varchar(50)  | NO   | UNI | NULL    |                |
| product_image_path | varchar(100) | NO   | UNI | NULL    |                |
| recommended_price  | int          | NO   |     | NULL    |                |
| description        | varchar(250) | NO   |     | NULL    |                |
+--------------------+--------------+------+-----+---------+----------------+
bash-4.4# mysql -uroot -pthisisatestthing. -DGiftcardSiteDB -e "describe LegacySite_card"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int          | NO   | PRI | NULL    | auto_increment |
| data       | longblob     | NO   |     | NULL    |                |
| amount     | int          | NO   |     | NULL    |                |
| fp         | varchar(100) | NO   | UNI | NULL    |                |
| used       | tinyint(1)   | NO   |     | NULL    |                |
| product_id | int          | NO   | MUL | NULL    |                |
| user_id    | int          | NO   | MUL | NULL    |                |
+------------+--------------+------+-----+---------+----------------+


```

This proves that kubernetes job for migration was successful in creating corresponding tables in database. But they are empty tables as we did not seed the database.

#### **Database Seeding**

I have created a kubernetes job django-seed.yaml in new directory k8MigrateSeed. This was not put in k8 as we do not want to run seeding when Django pod is created.

There are two new json files created to seed the database. They contain data same as products.csv and users.csv files but in json format for django models. Please note I have created these files manually but Django has a command (python manage.py dumpdata) to do this if needed. If Django has existing data in database, we can run this command to generate json file.

    GiftcardSite/LegacySite/fixtures/products.json
    GiftcardSite/LegacySite/fixtures/users.json

users.json file content:

```json
[
	{
		"model": "LegacySite.user",
		"pk": 6,
		"fields": {
		    "last_login": "2020-10-01 12:51:48.124599",
		    "username": "admin",
		    "password": "000000000000000000000000000078d2$18821d89de11ab18488fdc0a01f1ddf4d290e198b0f80cd4974fc031dc2615a3"
		}
	}
]
```

Please check products.json file for its contents.

This kubernetes job will run the below commands (please check GiftcardSite/k8MigrateSeed/django-seed.yaml for full job).

    python3 manage.py loaddata users.json
    python3 manage.py loaddata products.json
    
Please note users.json and products.json files are called fixtures. If they are present in fixtures directory of an installed App (in our case, LegacySite), Django knows where to get them and we do not need to specify the path. That is why I checked them in fixtures directory under LegacySite.

Data can be loaded/seeded by calling manage.py loaddata < fixturename >, where < fixturename > is the name of the fixture file. In this case, we have two fixtures (one for products and one for users) so kubernetes job for seeding will run loaddata for both of them.

Once the migration (mentioned earlier) is completed successfully, run the seeding job by running the below command.

    kubectl apply -f GiftcardSite/k8MigrateSeed/django-seed.yaml
    
Check the status too by running kubectl get pods.


```plaintext
appsecstudent@appsecstudent-VirtualBox:~/AppSecAssignment3/AppSecAssignment3$ kubectl apply -f GiftcardSite/k8MigrateSeed/django-seed.yaml 
job.batch/assignment3-django-seeding created
appsecstudent@appsecstudent-VirtualBox:~/AppSecAssignment3/AppSecAssignment3$ kubectl get pods
NAME                                         READY   STATUS      RESTARTS   AGE
assignment3-django-deploy-7cb68dc4cf-l2gnp   1/1     Running     0          6m12s
assignment3-django-migrations-snqzd          0/1     Completed   0          4m48s
assignment3-django-seeding-twsv9             0/1     Completed   0          12s
mysql-container-79c5db5bdf-kz8ls             1/1     Running     0          6m22s
proxy-84787764d7-xcj9t                       1/1     Running     0          6m3s
appsecstudent@appsecstudent-VirtualBox:~/AppSecAssignment3/AppSecAssignment3$ 

```

If status for assignment3-django-seeding is completed, it is successful. Get a BASH shell in my sql container and run SQL queries to verify table data.

```plaintext
appsecstudent@appsecstudent-VirtualBox:~/AppSecAssignment3/AppSecAssignment3$ kubectl exec --stdin --tty mysql-container-79c5db5bdf-kz8ls  -- bash
bash-4.4#  mysql -uroot -pthisisatestthing. -DGiftcardSiteDB -e "select * from LegacySite_product"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+------------------------+-----------------------+-------------------+-------------------------------------------------------------------------------------------------+
| product_id | product_name           | product_image_path    | recommended_price | description                                                                                     |
+------------+------------------------+-----------------------+-------------------+-------------------------------------------------------------------------------------------------+
|          1 | NYU Apparel Card       | /images/product_1.jpg |                95 | Use this card to buy NYU Clothing!                                                              |
|          2 | Tandon Food Court Card | /images/product_2.jpg |                30 | Use this card to buy food at the Tandon Food Court!                                             |
|          3 | Graduation Robe Card   | /images/product_3.jpg |               199 | Why worry about this later? Buy this card to make graduation easier!                            |
|          4 | Semester's Book Card   | /images/product_5.jpg |               777 | So much to read, so little time. Buy this to make payment at the book store quicker and easier! |
|          5 | NYU Electronics Card   | /images/product_6.jpg |               500 | Need a new laptop? No problem! This card can be used to buy electronics at the NYU Bookstore.   |
|          6 | Tuition Card           | /images/product_7.jpg |              1999 | Need to pay for those credits? Pick up this card to make the process easier!                    |
|          7 | NYU Gym Card           | /images/product_8.jpg |               450 | Want summer gym access for your entire degree? This card should cover it!                       |
+------------+------------------------+-----------------------+-------------------+-------------------------------------------------------------------------------------------------+
bash-4.4# mysql -uroot -pthisisatestthing. -DGiftcardSiteDB -e "select * from LegacySite_user"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------------------------+----------+---------------------------------------------------------------------------------------------------+
| id | last_login                 | username | password                                                                                          |
+----+----------------------------+----------+---------------------------------------------------------------------------------------------------+
|  6 | 2020-10-01 12:51:48.124599 | admin    | 000000000000000000000000000078d2$18821d89de11ab18488fdc0a01f1ddf4d290e198b0f80cd4974fc031dc2615a3 |
+----+----------------------------+----------+---------------------------------------------------------------------------------------------------+
```


After this, I ran the website and did some actions such as register a user, buy/use/gift a card. All worked perfectly fine.
 
yaml file for the migrations job - GiftcardSite/k8MigrateSeed/django-migrations.yaml
yaml file for the database seeding job - GiftcardSite/k8MigrateSeed/django-seed.yaml
No new Dockerfile is used for this part. db/Dockerfile was updated to remove SQL script.
No new code is created to perform database seeding. Two new json files with data are created and then loaded by using python3 manage.py loaddata.
 
