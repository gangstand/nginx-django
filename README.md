# Заходим через root пользователя
Обновляем и устанавливаем нужные пакеты для VDS сервера
```
apt-get update
apt-get install -y sudo vim mosh tmux htop git curl wget unzip zip gcc build-essential make vsftpd ufw
```
# Настраиваем FTP
```
systemctl start vsftpd
systemctl enable vsftpd
ufw allow 20
ufw allow 21
```
Открываем FTP конфиг
```
nano /etc/vsftpd.conf
```
Изменяем конфиг
```
listen=YES
#listen_ipv6=YES
write_enabl=YES
local_unmask=022
```
Перезапускаем FTP сервер
```
service vsftpd stop
service vsftpd start
```
# Пользователь
Устанавливаем пароль для пользоваля www (Для удобства такой же как и на root пользователя)
```
adduser www
```
Устанавливаем пароль для FTP пользоваля www (Для удобства такой же как и на root пользователя)
```
passwd www
```
Предоставляем фулл доступ для редактирование пользователю
```
chown www /home/www
```
Даём доступ в конфиге для www пользователя
```
nano /etc/sudoers
```
Добавить
```
www	ALL=(ALL)  ALL
```
## Заходим через www пользователя
Устанавливаем нужные пакеты для VDS сервера
```
sudo apt-get install -y zsh tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python-pil python3-lxml libxslt-dev python-libxml2 libffi-dev libssl-dev python-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor
```
Устанавливаем zch скриптовый интерпретатор
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
Настраиваем скриптовый интерпретатор для Python
```
nano ~/.zshrc
```
Добавить
```
export PATH=$PATH:/home/www/.python/bin
alias cls="clear"
```
Проверяем скриптовый интерпретатор
```
chsh -s $(which zsh)
which zsh
```
Скачиваем архив Python и собираем

```
wget https://www.python.org/ftp/python/3.10.1/Python-3.10.1.tgz
tar xvf Python-3.10.1
cd Python-3.10.1
mkdir ~/.python
./configure --enable-optimizations --prefix=/home/www/.python
make -j8
sudo make altinstall
. ~/.zshrc
```
Возвращаемcя на главную
```
cd
mkdir code
cd code
mkdir taskmanager
cd taskmanager
python3.10 -m venv env
. env/bin/activate
```
!!! ОБЯЗАТЕЛЬНО УСТАНОВИТЬ ДАННЫЙ ПАКЕТ !!!
```
pip install gunicorn
```
!!! ОБЯЗАТЕЛЬНО УСТАНОВИТЬ ДАННЫЙ ПАКЕТ !!!
# Перекидываем проект джанго
Устанавливаем нужные пакеты для проекта
```
pip install -r requirements.txt
```
Настраиваем settings
```
DEBUG = False

ALLOWED_HOSTS = ['127.0.0.1', 'домен', 'ip домена']

STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, "static")

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, "media")
```
Создать gunicorn_config.py по пути ```#pwd /home/www/code/taskmanager/taskmanager```
```
sudo nano gunicorn_config.py
```
Добавить
```
command = '/home/www/code/taskmanager/env/bin/gunicorn'
pythonpath = '/home/www/code/taskmanager/taskmanager'
bind = '127.0.0.1:8001'
workers = 3
user = 'www'
limit_request_fields = 32000
limit_request_field_size = 0
raw_env = 'DJANGO_SETTINGS_MODULE=taskmanager.settings'
```
Создать start_gunicorn.sh по пути ```#pwd /home/www/code/taskmanager/bin```
```
mkdir bin
nano bin/start_gunicorn.sh
```
Добавить
```
#!/bin/bash
source /home/www/code/taskmanager/env/bin/activate
exec gunicorn  -c "/home/www/code/taskmanager/taskmanager/gunicorn_config.py" taskmanager.wsgi
```
Выдаём права
```
chmod +x bin/start_gunicorn.sh
```
Настваиваем nginx
```
sudo rm -fr /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-enabled/default
sudo service nginx stop
```
Добавить
```
server {

    listen 80;
    server_name 149.154.64.149;
    
    client_max_body_size 4G;
    
    location /static/ {
        alias  /home/www/code/taskmanager/taskmanager/static/;
    }
    
    location /media/ {
        alias /home/www/code/taskmanager/taskmanager/media/;
    }
    
    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        
        if (!-f $request_filename) {
            proxy_pass http://127.0.0.1:8001;
            break;
        }

        error_page 500 502 503 504 /500.html;
        location = /500.html {
            root  /home/www/code/taskmanager/taskmanager/static/;
        }
    }
}

server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
        }
}

```
Настраиваем supervisor
```
sudo nano /etc/supervisor/conf.d/taskmanager.conf
```
Добавить
```
[program:www_gunicorn]
command=/home/www/code/taskmanager/bin/start_gunicorn.sh
user=www
process_name=%(program_name)s
numprocs=1
autostart=true
autorestart=true
redirect_stderr=true
```
Запуск сервера
```
sudo service nginx start
sudo service supervisor start
```
# Защита от Ddos
```
sudo rm -fr /etc/nginx/nginx.conf
sudo nano /etc/nginx/nginx.conf
```
Добавить
```
user www-data;
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 30000;

pcre_jit on;

pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 8192;
}

http {
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        reset_timedout_connection on;
        keepalive_timeout 300;
        keepalive_requests 10000;
        send_timeout 1200;
        client_body_timeout 30;
        client_header_timeout 30;
        types_hash_max_size 2048;
        server_names_hash_max_size      4096;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;


        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;


        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
        client_max_body_size       10m;

        proxy_connect_timeout      5;
        proxy_send_timeout         10;
        proxy_read_timeout         10;
        proxy_temp_file_write_size 64k;
        proxy_buffer_size          4k;
        proxy_buffers              32 16k;
        proxy_busy_buffers_size    32k;



        gzip            on;
        gzip_static             on;
        gzip_types              text/plain text/css text/xml application/javascript application/json application/msword application/rtf application/pdf application/vnd$
        gzip_comp_level 7;
        gzip_proxied    any;
        gzip_min_length 1000;
        gzip_disable    "msie6";
        gzip_vary       on;

        etag    off;

        open_file_cache          max=10000 inactive=60s;
        open_file_cache_valid    30s;
        open_file_cache_errors   on;
        open_file_cache_min_uses 2;

        proxy_cache_valid 1h;
        proxy_cache_key $scheme$proxy_host$request_uri$cookie_US;
        limit_conn_zone $binary_remote_addr$host zone=lone:10m;
        limit_req_zone  $binary_remote_addr$host zone=ltwo:10m   rate=3r/s;
        limit_req_zone  $binary_remote_addr$host zone=highspeed:10m  rate=20r/s;

        log_format postdata '$remote_addr - $time_local - $request_body';

        map $http_accept $webp_suffix {
        "~*webp"  ".webp";
        }

        map $msie $cache_control {
            default "max-age=31536000, public, no-transform, immutable";
	"1"     "max-age=31536000, private, no-transform, immutable";
        }

        map $msie $vary_header {
        default "Accept";
        "1"     "";
        }

        map $http_user_agent $limit_bots {
        default 0;
        ~*(google|bing|yandex|msnbot) 1;
        ~*(AltaVista|Googlebot|Slurp|BlackWidow|Bot|ChinaClaw|Custo|DISCo|Download|Demon|eCatch|EirGrabber|EmailSiphon|EmailWolf|SuperHTTP|Surfbot|WebWhacker) 1;
        ~*(Express|WebPictures|ExtractorPro|EyeNetIE|FlashGet|GetRight|GetWeb!|Go!Zilla|Go-Ahead-Got-It|GrabNet|Grafula|HMView|Go!Zilla|Go-Ahead-Got-It) 1;
        ~*(rafula|HMView|HTTrack|Stripper|Sucker|Indy|InterGET|Ninja|JetCar|Spider|larbin|LeechFTP|Downloader|tool|Navroad|NearSite|NetAnts|tAkeOut|WWWOFFLE) 1;
        ~*(GrabNet|NetSpider|Vampire|NetZIP|Octopus|Offline|PageGrabber|Foto|pavuk|pcBrowser|RealDownload|ReGet|SiteSnagger|SmartDownload|SuperBot|WebSpider) 1;
        ~*(Teleport|VoidEYE|Collector|WebAuto|WebCopier|WebFetch|WebGo|WebLeacher|WebReaper|WebSauger|eXtractor|Quester|WebStripper|WebZIP|Wget|Widow|Zeus) 1;
        ~*(Twengabot|htmlparser|libwww|Python|perl|urllib|scan|Curl|email|PycURL|Pyth|PyQ|WebCollector|WebCopy|webcraw) 1;
        }

        include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}

```
