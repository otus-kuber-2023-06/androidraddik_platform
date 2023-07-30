Домашнее задание 9

Запущен кластер minikube

minikube start  

Создан Custom resource мафнифест kubernetes-operators\deploy\cr.yml:

kubectl apply -f cr.yml

error: unable to recognize "deploy/cr.yml": no matches for kind "MySQL" in version "otus.homework/v1"

Ошибка связана с отсутсвием объектов типа MySQL в API kubernetes. Необходимо определить  custom resorce. Для этого создан Custom Resouce Definition (CRD):

kubectl apply -f crd.yml

Применение манифеста:

kubectl apply -f cr.yml

Информация:

kubectl get crd
NAME                   CREATED AT
mysqls.otus.homework   2022-11-24T17:34:27Z

kubectl get mysqls.otus.homework
NAME             AGE
mysql-instance   19s

kubectl describe mysqls.otus.homework mysql-instance

Name:         mysql-instance
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  otus.homework/v1
Kind:         MySQL
Metadata:
  Creation Timestamp:  2022-11-24T17:35:21Z
  Generation:          1
  Managed Fields:
    API Version:  otus.homework/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:database:
        f:image:
        f:password:
        f:storage_size:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2022-11-24T17:35:21Z
  Resource Version:  1034
  UID:               637ab76e-9067-463f-bbf8-8ba74966b74f
Spec:
  Database:      otus-database
  Image:         mysql:5.7
  Password:      otuspassword
  storage_size:  1Gi
Events:          <none>


Добавлена Validation в спецификацию CDR, применены манифесты crd.yml и cr.yml:

kubectl delete mysqls.otus.homework mysql-instance
kubectl apply -f crd.yml
kubectl apply -f cr.yml

Объявлены "обязательне" поля в спецификации с помощью дерективу spec.validation.spec.reqiored в CRD. 

Удаление "обязательного" поля из манифеста:

kubectl apply -f deploy/crd.yml
customresourcedefinition.apiextensions.k8s.io/mysqls.otus.homework configured

kubectl apply -f deploy/cr.yml
The MySQL "mysql-instance" is invalid: spec.storage_size: Required value

Создана функция для создания объекта myqsql:

kopf run msql-operator.py 

[2022-11-29 17:51:00,329] kopf.objects         [INFO    ] [default/mysql-instance] Handler 'mysql_on_create' succeeded.
[2022-11-29 17:51:00,329] kopf.objects         [INFO    ] [default/mysql-instance] Creation is processed: 1 succeeded; 0 failed.

Вопрос: "Почему объект создался, хотя мы создали CR, до того, как запустили контроллер?"

Запись о CustomResource попала в etcd и информация о ней может быть получена через kube-apiserver. Когда появляется новый ресурс, его обнаруживает контроллер mysql-operator, в задачи которого входит отслеживание изменений среди соответствующих записей пкесурсов (MySql). В данном случае контроллер регистрирует специальный callback для событий создания через информатор. Этот обработчик будет вызван, когда mysql-operator впервые станет доступным, и начнёт свою работу с добавления объекта во внутреннюю очередь. К тому времени, когда он дойдёт до обработки этого объекта, контроллер проверит и поймёт, что нет связанных с ним записей pv, pvc, service, deployment и подов. Эту информацию он получает, опрашивая kube-apiserver по label selectors. Важно заметить, что этот процесс синхронизации ничего не знает о состоянии (является state agnostic): он проверяет новые записи точно так же, как и уже существующие. А значит не важно когда контроллер был запущен, он запросит и обработает все записи о ресурсах, за кторые он отвечает и только для тех которые ещё "не обработаны" юужут созданы соответствующие ресурсы.

Удаление всех ресурсов, созданных контроллером:

kubectl delete mysqls.otus.homework mysql-instance
kubectl delete deployments.apps mysql-instance
kubectl delete pvc mysql-instance-pvc
kubectl delete pv mysql-instance-pv
kubectl delete svc mysql-instance

Добавление обработки удаления объектов в код согласно примеру:

kopf run mysql-operator.py

Добавление создания и воссатновления из бекапов ресурса согласно примеру с добавлением строки kopf.append_owner_reference(restore_job, owner=body)

Проверка:

kopf run mysql-operator.py

kubectl apply -f cr.yml
kubectl get pvc


kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key (id) );" otus-database

kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data' );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data-2' );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database


Удаление mysql-instance:

kubectl delete mysqls.otus.homework mysql-instance
kubectl get pv
kubectl get jobs.batch


Создадние новой копии инстанса и проверка получилось ли восстановится из backup:

kubectl apply -f cr.yml

export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database

Контроллер работает. 

cd build
docker build -t oksanazh/mysql-operator:v0.1 
docker push  oksanazh/mysql-operator:v0.1 

Создание роли, sa и deployment и загрузка контроллер в кластер:

kubectl apply -f /deploy

Проверка:

kubectl apply -f cr.yml
kubectl get pvc
export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")

kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key (id) );" otus-database

kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data' );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data-2' );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database

Удаление mysql-instance:

kubectl delete mysqls.otus.homework mysql-instance
kubectl get pv
kubectl get jobs.batch

Создание новой копии  инстанса и проверка восстановления из backup:

kubectl apply -f cr.yml

export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+


kubectl get jobs
NAME                         COMPLETIONS   DURATION   AGE
backup-mysql-instance-job    1/1           5s         5m21s
restore-mysql-instance-job   1/1           54s        5m32s