** Step 1
Setting up the S3 replacement minio storage, it is hosted locally
```
cd minio_app
kubectl create ns minio
kubens minio
kubectl create -f minio.yaml
kubectl port-forward -n minio svc/minio 9000:9000 
http://localhost:9000
user: minio
paswd: minio123

create a bucket kanister for later use
```

** Step 2
Setting up the mysql database
```
cd mysql_app
kubens default

# Create the mysql secret
kubectl create secret generic mysql \
  --from-literal=mysql-root-password='password'

kubectl create -f mysql.yaml

kubectl run mysql-client --image=mysql:5.7 -it --rm --restart=Never -- mysql -h mysql -uroot -ppassword -e 'SELECT 1'

kubectl exec -ti mysql-0 -- bash
root@mysql-0:/# mysql -p    

...
mysql> CREATE DATABASE test;
mysql> USE test;
mysql> CREATE TABLE pets (name VARCHAR(20), species VARCHAR(20));
mysql> INSERT INTO pets VALUES ('flare', 'dog');
mysql> INSERT INTO pets VALUES ('reno', 'dog');
mysql> SELECT * FROM pets;
+-------+---------+
| name  | species |
+-------+---------+
| bingo | dog     |
| kalu  | dog     |
+-------+---------+
2 rows in set (0.00 sec)

mysql> exit
root@mysql-0:/# exit

```

** Settting up Kanister
```
kubectl create ns kanister
kubens kanister

helm repo add kanister http://charts.kanister.io

helm install myrelease --namespace kanister kanister/kanister-operator --set image.tag=0.44.0

kubectl get deploy,pod
kubectl get crd |grep kanister

```

** Playing with CRD
```
cd PROJECT_ROOT

kubectl create secret -n kanister generic minio-secret \
 --from-literal=minio_access_key_id='minio' \
 --from-literal=minio_secret_access_key='minio123'

kubectl apply -f profile.yaml

Getting the blueprint
git clone git@github.com:kanisterio/kanister.git
kubectl apply -f kanister/examples/stable/mysql/mysql-blueprint.yaml 
kubectl get blueprint
```

** Backup
```
kubectl create -f backup.yaml

kubectl describe actionset -n kanister backup 
...
Events:
  Type    Reason           Age    From                 Message
  ----    ------           ----   ----                 -------
  Normal  Started Action   3m18s  Kanister Controller  Executing action backup
  Normal  Started Phase    3m18s  Kanister Controller  Executing phase dumpToObjectStore
  Normal  Ended Phase      35s    Kanister Controller  Completed phase dumpToObjectStore
  Normal  Update Complete  35s    Kanister Controller  Updated ActionSet 'backup' Status->complete

  **This may take some time, depends on size of dump

kubectl get actionsets backup -o yaml |grep -A 3 artifacts
get the s3path: /mysql-backups/default/mysql/2020-12-12T08-26-08/dump.sql.gz
```

** Disaster strikes!
```
kubectl exec -ti mysql-0 -- bash
root@mysql-0:/# mysql -p    

mysql> SHOW DATABASES;
DROP DATABASE test;
SHOW DATABASES;

```

** Restore the Application
```
put the s3path in restore.yaml

kubectl create -f restore.yaml
kubectl describe actionset -n kanister restore
```



** Verify
```
kubens default
kubectl exec -ti mysql-0 -- bash
mysql -p
mysql> SHOW DATABASES;
Your dropped tables should be here

```
