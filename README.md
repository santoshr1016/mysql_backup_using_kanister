** Step 1
Setting up the S3 replacement minio storage, it is hosted locally
```
cd minio_app
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
kubectl create -f mysql.yaml
kubectl -n default exec -it mysql-0 -- bash
root@mysql-0:/# mysql -p    

...
mysql> CREATE DATABASE test;
mysql> USE test;
mysql> CREATE TABLE pets (name VARCHAR(20), species VARCHAR(20));
mysql> INSERT INTO pets VALUES ('bingo', 'dog');
mysql> INSERT INTO pets VALUES ('kalu', 'dog');
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

** Settting up Kanister CRD
```
cd kanister
kubectl apply -f minio-secret.yaml

helm repo add kanister http://charts.kanister.io
helm repo update

helm install myrelease --namespace kanister kanister/kanister-operator --set image.tag=0.44.0

kubectl -n kanister get deploy,pod
kubectl -n kanister get crd |grep kanister

** Important APPLY the mysql profile
kubectl apply -f profile.yaml
```

** How to get mysql profile
```
Getting the blueprint

git clone git@github.com:kanisterio/kanister.git
kubectl apply -f kanister/examples/stable/mysql/mysql-blueprint.yaml 
kubectl get blueprint
```

** Operations  Backup action
```
cd operations
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

Note: Everytime you run backup, change the name of new backup in the yaml
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
like this s3path: /mysql-backups/default/mysql/2020-12-12T08-26-08/dump.sql.gz

get this from kubectl describe actionset -n kanister backup 

kubectl create -f restore.yaml
kubectl describe actionset -n kanister restore

Note: Everytime you run restore, change the name of the new restore in yaml
```



** Verify
```
kubens default
kubectl exec -ti mysql-0 -- bash
mysql -p
mysql> SHOW DATABASES;
Your dropped tables should be here

```

** Very Important.
** Want to take backup again, delete the old actionset and re-apply
```
➜  operations git:(main) ✗ k get actionsets.cr.kanister.io      
NAME      AGE
backup    6m16s
restore   118s

➜  operations git:(main) ✗ k delete actionsets.cr.kanister.io backup
➜  operations git:(main) ✗ k delete actionsets.cr.kanister.io restore

➜  operations git:(main) ✗ k apply -f backup.yaml 
➜  operations git:(main) ✗ k apply -f restore.yaml 
```
