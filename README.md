# otus-hw6-mlflow


Запуск:

1. terraform apply -auto-approve

2. После разворачивания. Заходим на vm с Mlflow:

ssh -i /home/ubuntu/yc ubuntu@xx.xxx.xxx.xxx

Там поднимаем screen
(проверить screen -ls)

После чего поднимаем mlflow:
mlflow server --backend-store-uri postgresql://${DB_USER}:${DB_PASS}@${DB_HOST}:${DB_PORT}/${DB_NAME}?sslmode=verify-full --default-artifact-root s3://${S3_BUCKET}/artifacts -h 0.0.0.0 -p 8000

Проверить сертификат (права должны быть ubuntu)
Посмотреть файлы с правами: ls -l

Вход в UI: ip_внешний:8000

3. перекидываем файлы в s3 и даги на vm с airflow (сделать заранее папки src)

Это с базовой VM

make upload-dags-to-airflow
make upload-src-to-bucket

4. Зайти на airflow:

ssh -i /home/ubuntu/yc ubuntu@xx.xxx.xxx.xxx

Вход в UI: ip_внешний
логин: test_admin
пароль: видно при заходи на VM

5. сохранить окружение в архив:

на мастер ноде кластера:

python -m venv pyspark_venv && \
source pyspark_venv/bin/activate

pip install venv-pack loguru pandas hyperopt mlflow argparse
pip install venv-pack loguru pandas hyperopt mlflow argparse scipy

venv-pack -o hyp_mlf_pd_log_arg.tar.gz
venv-pack -o hyp_mlf_pd_log_arg_sc.tar.gz

hdfs dfs -copyFromLocal hyp_mlf_pd_log_arg.tar.gz s3a://cold-s3-bucket/venvs/
hdfs dfs -copyFromLocal hyp_mlf_pd_log_arg_sc.tar.gz s3a://cold-s3-bucket/venvs/

6. Чтобы засетапить .sh через спарк submit:

spark-submit --conf spark.yarn-appMasterEnv.PYSPARK_PYTHON=./venv/bin/python --conf spark.yarn.appMasterEnv.PYSPARK_DRIVER_PYTHON=./venv/bin/python --conf spark.yarn.dist.archives=s3a://cold-s3-bucket/venvs/hyp_mlfl_pand_log.tar.gz#venv --deploy-mode=cluster model.py

(убрать find_spark)

7. вручную добавить сертификат на postgree

mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
    --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt

8. вручную добавить конфиги для s3

touch .s3cfg в корне и пишем:
vim .s3cfg
[default]
access_key = XXXXX
secret_key = XXXXX
bucket_location = ru-central1
host_base = storage.yandexcloud.net
host_bucket = %(bucket)s.storage.yandexcloud.net

9. проверить файлы на s3

s3cmd ls s3://airflow-bucket-245a7f0181cac109
s3cmd ls s3://cold-s3-bucket

10. Разобрать все

terraform destroy -auto-approve

11. ПОМНИТЬ:

спарк конфиги добавляются только в properties в даге. если их добавить в коде - все падает
Убрать find spark из кода для запуска в даге. Иначе все упадет.
