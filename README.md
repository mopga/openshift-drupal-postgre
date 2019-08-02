Часть I, докерная.
==================

Для начала нужно подготовить Докерфайлы и запушить их в кастомный реджестри или на докерхаб.

Например:

```
echo '
FROM nginx:1.15
RUN useradd -u 1001 -r -g 0 -d /app -s /sbin/nologin -c "Default Application User" default \
&& mkdir -p /app \
&& chown -R 1001:0 /app && chmod -R g+rwX /app
COPY nginx.conf /etc/nginx
COPY drupal.conf /etc/nginx/conf.d/default.conf
RUN chown -R 1001:0 /var/log && chmod -R g+rwX /var/log
RUN chown -R 1001:0 /var/cache/nginx && chmod -R g+rwX /var/cache/nginx
RUN chown -R 1001:0 /var/run && chmod -R g+rwX /var/run
RUN chown -R 1001:0 /etc/nginx && chmod -R g+rwX /etc/nginx
EXPOSE 8080
USER 1001' > Dockerfile.nginx
```

--------------------


```
echo '
FROM php:7.1-fpm
RUN apt-get update \
&& apt-get install -y libfreetype6-dev libjpeg62-turbo-dev libpng-dev wget git libpq-dev
RUN docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
&& docker-php-ext-configure pgsql -with-pgsql=/usr/local/pgsql \
&& docker-php-ext-install gd \
&& :\
&& docker-php-ext-install pdo pdo_pgsql opcache zip \
&& docker-php-ext-enable pdo pdo_pgsql opcache zip
# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
RUN set -ex; \
\
# Drush
composer global require drush/drush:^8.0; \
\
# Drush launcher
wget -O drush.phar \
"https://github.com/drush-ops/drush-launcher/releases/download/0.6.0/drush.phar"; \
chmod +x drush.phar; \
mv drush.phar /usr/local/bin/drush; \
\
# Drupal console
curl https://drupalconsole.com/installer -L -o drupal.phar; \
mv drupal.phar /usr/local/bin/drupal; \
chmod +x /usr/local/bin/drupal; \
\
# Clean up
composer clear-cache;
# /app will map to nginx container
RUN useradd -u 1001 -r -g 0 -d /app -s /sbin/nologin -c "Default Application User" default \
&& mkdir -p /app \
&& chown -R 1001:0 /app && chmod -R g+rwX /app
# This is where the source code will be cloned.
RUN mkdir /code && chown -R 1001:0 /code && chmod -R g+rwX /code
# Drupal files directory will map to this.
RUN mkdir /shared && chown -R 1001:0 /shared && chmod -R g+rwX /shared
USER 1001
WORKDIR /app
RUN git clone --depth=1 https://github.com/mopga/drupal-8-composer.git /code
RUN cd /code && rm -rf .git && composer install' > Dockerfile.drupal
```

---------------

```
docker build -t mopga/openshift-nginx:1.0 -f Dockerfile.nginx .
docker push mopga/openshift-nginx:1.0
docker build -t mopga/openshift-drupal:1.0 -f Dockerfile.drupal .
docker push mopga/openshift-drupal:1.0
```



Часть II, OpenShift'овая.
=========================

Для установки Dev-окружения я использовал OKD - это такая бесплатная реализация Openshift с поддержкой от самих RedHat.
Качаем [отсюда](https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz)

рапаковываем, кидаем в нужное место и делаем запускаемым:

```
tar -xzf openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz
cd openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit/
cp * /usr/local/bin/
chmod +x /usr/local/bin/oc
chmod +x /usr/local/bin/kubectl
```

Тепроь прсото делаем в какой угодно папке (хоть в хомяке) :

```
oc cluster up
```

Дожидаемся пока он скачает контейнеры и все. Веб-морда доступна на https://localhost:8443

---------------
  * Зайти под пользователем, создать новый проект:

```
    oc login -u developer
    oc new-project drupal-8
```

---------------

  * Делаем импорт наших образов и задаем им тег если нужно (чтобы опеншифт мог использовать их для созадния POD с нашим приложением и реагировать на изменеия в коде):

```
oc import-image openshift-drupal:1.0 --from=mopga/openshift-drupal:1.0 --confirm
oc import-image openshift-nginx:1.0 --from=mopga/openshift-nginx:1.0 --confirm
oc tag openshift-drupal:1.0 openshift-drupal:latest
oc tag openshift-nginx:1.0 openshift-nginx:latest
```

---------------

  * Создать секреты для проекта (например логин и пароль в базу, токен для доступа и т.д.):

```
kind: Secret
apiVersion: v1
metadata:
  name: drupal-8
  labels:
    app: drupal-8
stringData:
  database-user: drupal8
  database-password: dDBbhf372yig1
```

и применить файлик через:

```
    oc apply -f db_secret.yml
```

---------------

  * Запросить постоянное хранилище (Volume как в докере) для базы, где будут лежать данные ,которые должны быть персистентны:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: drupal-8-db
  labels:
    app: drupal-8
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

и применяем:

```
    oc apply -f db_volume.yml
```

---------------

  * Создаем deploymentConfig для базы, где указываем, ЧТО именно, КАК разворачивать, как проверять что развернулось:

```
apiVersion: v1
kind: DeploymentConfig
metadata:
  name: drupal-8-db
  labels:
    app: drupal-8  
spec:
  replicas: 1
  selector:
    name: drupal-8-db
    app: drupal-8    
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        name: drupal-8-db
        app: drupal-8  
    spec:
      containers:
      - env:
        - name: POSTGRESQL_USER
          valueFrom:
            secretKeyRef:
              key: database-user
              name: drupal-8
        - name: POSTGRESQL_PASSWORD
          valueFrom:
            secretKeyRef:
              key: database-password
              name: drupal-8
        - name: POSTGRESQL_DATABASE
          value: drupal8
        image: ' '
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 30
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 5432
          timeoutSeconds: 1
        name: postgres
        ports:
        - containerPort: 5432
          protocol: TCP

        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/lib/postgresql/data/
          name: drupal-8-db-data
      volumes:
      - name: drupal-8-db-data
        persistentVolumeClaim:
          claimName: drupal-8-db
  triggers:
  - imageChangeParams:
      automatic: true
      containerNames:
      - postgres
      from:
        kind: ImageStreamTag
        name: postgresql:10
        namespace: openshift
    type: ImageChange
  - type: ConfigChange
```

и применяем его:

```
    oc apply -f db_deploy.yml
```

---------------

  * Теперь нужно сделать чтобы наш постгрес был виден приложению, для этого создадим описание сервиса :

```
apiVersion: v1
kind: Service
metadata:
  name: drupal-8-db
  labels:
    app: drupal-8
spec:
  ports:
  - name: postgres
    port: 5432
    protocol: TCP
    targetPort: 5432
  selector:
    name: drupal-8-db
    app: drupal-8
```

и тоже применяем:

```
    oc apply -f db_service.yml
```

---------------

  * Можно наконец приступить к деплою нашего приложения. Пишем деплой конфиг теперь уже для него, так же как делали для базы:


  сначала создадим хранилище, так же как и для БД:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: drupal-8-db
  labels:
    app: drupal-8
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
  и применим его:

```
oc apply -f app_volume.yml
```

  а теперь сам DeploymentConfig:

```
apiVersion: v1
kind: DeploymentConfig
metadata:
  name: drupal-8
  labels:
    app: drupal-8
spec:
  replicas: 1
  selector:
    app: drupal-8
  template:
    metadata:
      labels:
        app: drupal-8
    spec:
      volumes:
        # Create the shared files volume to be used in both pods
        - name: app
          emptyDir: {}
        - name: drupal-8-files
          persistentVolumeClaim:
            claimName: drupal-8-files
      containers:
      - name: php-fpm
        image: 'mopga/openshift-drupal:1.0'
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              key: database-user
              name: drupal-8
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              key: database-password
              name: drupal-8
        - name: PG_HOST
          value: drupal-8-db
        - name: PG_PORT
          value: "5432"
        - name: POSTGRES_DB
          value: drupal8
        - name: OPENSHIFT_BUILD_NAME
          value: "1"
        volumeMounts:
          - name: app
            mountPath: /app
          - name: drupal-8-files
            mountPath: /shared
        lifecycle:
          postStart:
            exec:
              command:
                - "/bin/sh"
                - "-c"
                - > 
                  cp -fr /code/. /app;
                  rm -rf /app/web/sites/default/files;
                  ln -s /shared /app/web/sites/default/files;
      - name: nginx
        image: 'mopga/openshift-nginx:1.0'
        ports:
          - name: http
            containerPort: 8080
        volumeMounts:
          - name: app
            mountPath: /app
          - name: drupal-8-files
            mountPath: /shared
  triggers:
    - type: ConfigChange
    - type: ImageChange
      imageChangeParams:
        automatic: true
        containerNames:
          - php-fpm
        from:
          kind: "ImageStreamTag"
          name: "openshift-drupal:latest"
          namespace: drupal-8
    - type: ImageChange
      imageChangeParams:
        automatic: true
        containerNames:
          - nginx
        from:
          kind: "ImageStreamTag"
          name: "openshift-nginx:latest"
          namespace: drupal-8
```

  применяем:

```
oc apply -f app_deploy.yml
```

---------------
  * Теперь нам нужно опубликовать сервис:

```
kind: Service
apiVersion: v1
metadata:
  name: drupal-8
  labels:
    app: drupal-8
spec:
  selector:
    app: drupal-8
  ports:
    - name: http
      port: 8080
      protocol: TCP
```

```
oc apply -f app_service.yml
```

  * Делаем его доступным снаружи:

```
    oc expose svc/drupal-8
```

  проверяем, что создался маршрут:
   
``` 
    oc get routes:
```
 
> drupal-8-drupal-8.127.0.0.1.nip.io

  * Попробуем обновить наше приложение. Мне например потребовалось изменить настройки Ngixnx. 

  Собираем новый образ и пушим в реджистри:

```
    docker build -t mopga/openshift-nginx:1.1 -f Dockerfile.nginx .
    docker push mopga/openshift-nginx:1.1
```

Импортим новый образ в опеншифт:

```
    oc import-image openshift-nginx:1.1 --from=mopga/openshift-nginx:1.1 --confirm
```

и делаем его актуальным:

```
    oc tag openshift-nginx:1.1 openshift-nginx:latest
```
  * ПРоверяем наш сайт - drupal-8-drupal-8.127.0.0.1.nip.io. Все должно работать успешно.

