### OTUS Linux Professional Lesson #10

#### ЦЕЛЬ:
Управление пакетами. Дистрибьюция софта
#### ОПИСАНИЕ ДОМАШНЕГО ЗАДАНИЯ:
Основная часть: 
* Создать свой RPM пакет
* Создать свой репозиторий и разместить там ранее собранный RPM
* Реализовать это все либо в Vagrant, либо развернуть у себя через Nginx и дать ссылку на репозиторий

>[!NOTE]
> Реализация задания основана на выполнении сценария config.sh в блоке provision vagrant-файла

#### ВЫПОЛНЕНИЕ:
Разворачивание вирутальной машины:
Используя Vagrantfile создаем вирутальную машину на основе Almalinux

#### 1. Создание RPM пакета:
   
Устанавливаем необходимые пакеты:
```
yum install -y wget rpmdevtools rpm-build createrepo yum-utils cmake gcc git nano
```
Собираем пакет Nginx c дополнительным модулем ngx_brotli
Загружаем SRPM пакет Nginx для дальнейшей работы над ним:
```
mkdir rpm && cd rpm
yumdownloader --source nginx
```
Устанавливаем зависимости для сборки пакета Nginx:
```
rpm -Uvh nginx*.src.rpm
yum-builddep nginx
```
Скачиваем исходный код модуля ngx_brotli:
```
cd /root
git clone --recurse-submodules -j8 \https://github.com/google/ngx_brotli
cd ngx_brotli/deps/brotli
mkdir out && cd out
```
Собираем модуль ngx_brotli:
```
cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..
cmake --build . --config Release -j 2 --target brotlienc
cd ../../../..
```
Редактируем spec-фаил,добавим указание на модуль --add-module=/root/ngx_brotli \ :
cd ~/rpmbuild/SPECS/
nano nginx.spec
```
if ! ./configure \
    --prefix=%{_datadir}/nginx \
    --sbin-path=%{_sbindir}/nginx \
    --modules-path=%{nginx_moduledir} \
    --conf-path=%{_sysconfdir}/nginx/nginx.conf \
    --error-log-path=%{_localstatedir}/log/nginx/error.log \
    --http-log-path=%{_localstatedir}/log/nginx/access.log \
    --http-client-body-temp-path=%{_localstatedir}/lib/nginx/tmp/client_body \
    --http-proxy-temp-path=%{_localstatedir}/lib/nginx/tmp/proxy \
    --http-fastcgi-temp-path=%{_localstatedir}/lib/nginx/tmp/fastcgi \
    --http-uwsgi-temp-path=%{_localstatedir}/lib/nginx/tmp/uwsgi \
    --http-scgi-temp-path=%{_localstatedir}/lib/nginx/tmp/scgi \
    --pid-path=/run/nginx.pid \
    --lock-path=/run/lock/subsys/nginx \
    --user=%{nginx_user} \
    --group=%{nginx_user} \
    --with-compat \
    --with-debug \
    --add-module=/root/ngx_brotli \
```
Приступаем к сборке RPM пакета:
```
rpmbuild -ba nginx.spec -D 'debug_package %{nil}'
```
Проверим содержимое директории:
```
[vagrant@rpm ~]$  ll /root/rpmbuild/RPMS/x86_64
total 2008
-rw-r--r--. 1 root root   36353 Aug 30 08:07 nginx-1.20.1-14.el9.1.alma.1.x86_64.rpm
-rw-r--r--. 1 root root    7485 Aug 30 08:07 nginx-all-modules-1.20.1-14.el9.1.alma.1.noarch.rpm
-rw-r--r--. 1 root root 1021307 Aug 30 08:07 nginx-core-1.20.1-14.el9.1.alma.1.x86_64.rpm
-rw-r--r--. 1 root root    8555 Aug 30 08:07 nginx-filesystem-1.20.1-14.el9.1.alma.1.noarch.rpm
-rw-r--r--. 1 root root  759157 Aug 30 08:07 nginx-mod-devel-1.20.1-14.el9.1.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   19498 Aug 30 08:07 nginx-mod-http-image-filter-1.20.1-14.el9.1.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   30996 Aug 30 08:07 nginx-mod-http-perl-1.20.1-14.el9.1.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   18282 Aug 30 08:07 nginx-mod-http-xslt-filter-1.20.1-14.el9.1.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   53927 Aug 30 08:07 nginx-mod-mail-1.20.1-14.el9.1.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   80551 Aug 30 08:07 nginx-mod-stream-1.20.1-14.el9.1.alma.1.x86_64.rpm
```
Копируем содержимое в общий каталог:
```
[root@repo x86_64]# cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/
[root@repo x86_64]# cd ~/rpmbuild/RPMS/x86_64
```
Устанавливаем пакет и проверим nginx:
```
yum localinstall *.rpm
```
```
systemctl start nginx
systemctl status nginx
```
```
[vagrant@rpm ~]$ service nginx status
Redirecting to /bin/systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Fri 2024-08-30 08:07:56 UTC; 10min ago
   Main PID: 36945 (nginx)
      Tasks: 2 (limit: 5572)
     Memory: 7.0M
        CPU: 38ms
     CGroup: /system.slice/nginx.service
             ├─36945 "nginx: master process /usr/sbin/nginx"
             └─37684 "nginx: worker process"
```
#### 2. Создание своего репозитория и размещение в нём ранее собранный RPM:
Создаём каталог repo:
```
mkdir /usr/share/nginx/html/repo
```
Копируем туда собранные RPM-пакеты:
```
cp ~/rpmbuild/RPMS/x86_64/*.rpm /usr/share/nginx/html/repo/
```
Инициализируем репозиторий командой:
```
createrepo /usr/share/nginx/html/repo/
```
```
Directory walk started
Directory walk done - 10 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```
Настроим в NGINX доступ к листингу каталога. В файле /etc/nginx/nginx.conf
в блоке server добавим следующие директивы:
index index.html index.htm;
autoindex on;
```
include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        index index.html index.htm;
        autoindex on;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

```
Проверяем синтаксис и перезапускаем NGINX:
```
nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
nginx -s reload
```
Протестируем репозиторий, добавим его в /etc/yum.repos.d:
```
cat >> /etc/yum.repos.d/otus.repo << EOF
> [otus]                                  
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF
```
Проверим его содержимое:
```
yum repolist enabled | grep otus

otus                             otus-linux
```
Добавим пакет в репозиторий:
```
cd /usr/share/nginx/html/repo/
wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm
```
Обновим список пакетов в репозитории:
```
createrepo /usr/share/nginx/html/repo/
```
```
Directory walk started
Directory walk done - 11 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```
```
yum makecache
```
Установим репозиторий percona-release:
```
yum install -y percona-release.noarch
```
