Deploying to production
=======================
Linux Mint 19 (Ubuntu bionic)
-----------------------------
 Installing nginx
  $sudo apt install nginx
 Installing uwsgi (on virtualenv django_ocr_server)
  $pip install uwsgi
 Create {path_to_your_project}/uwsgi.ini
  .. code-block::

   [uwsgi]
   chdir = {path_to_your_project}  # e.g. /home/shmakovpn/ocr_server
   module = {your_project}.wsgi  # e.g. ocr_server.wsgi
   home = {path_to_your_virtualenv}  # e.g. /home/shmakovpn/.virtualenvs/django_ocr_server
   master = true
   processes = 10
   http = 127.0.0.1:8003
   vacuum = true

 Create /etc/nginx/sites-available/django_ocr_server.conf
  .. code-block::

   server {
       listen 80;  # choose port what you want
       server_name _;
       charset utf-8;
       client_max_body_size 75M;
       location /static/rest_framework_swagger {
           alias {path_to_your virtualenv}/lib/python3.6/site-packages/rest_framework_swagger/static/rest_framework_swagger;
       }
       location /static/rest_framework {
            alias {path_to_your virtualenv}/lib/python3.7/site-packages/rest_framework/static/rest_framework;
       }
       location /static/admin {
           alias {path_to_your virtualenv}/lib/python3.7/site-packages/django/contrib/admin/static/admin;
       }
       location / {
           proxy_pass http://127.0.0.1:8003;
       }
   }

  Enable the django_ocr_server site
   $sudo ln -s /etc/nginx/sites-available/django_ocr_server.conf /etc/nginx/sites-enabled/

  Remove the nginx default site
   $sudo rm /etc/nginx/sites-enabled/default

  Create the systemd service unit /etc/systemd/system/django-ocr-server.service
   .. code-block::

    [Unit]
    Description=uWSGI Django OCR Server
    After=syslog.service

    [Service]
    User={your user}
    Group={your group}
    Environment="PATH={path_to_your_virtualenv}/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ExecStart={path_to_your_virtualenv}/bin/uwsgi --ini {path_to_your_project}/uwsgi.ini
    RuntimeDirectory=uwsgi
    Restart=always
    KillSignal=SIGQUIT
    Type=notify
    StandardError=syslog
    NotifyAccess=all

    [Install]
    WantedBy=multi-user.target

  Reload systemd
   $sudo systemctl daemon-reload
  Start the django-ocr-server service
   $sudo systemctl start django-ocr-server
  Enable the django-ocr-server service to start automatically after server is booted
   $sudo systemclt enable django-ocr-server
  Start nginx
   $sudo systemctl start nginx
  Enable nginx service to start automatically after server is booted
   $sudo systemctl enable nginx
  Go to http://{your_server}:80
   You will be redirected to admin page

Centos 7
--------
 Installing nginx
  $sudo apt install nginx
 Installing uwsgi (on virtualenv django_ocr_server)
  $pip install uwsgi
 Create /var/www/ocr_server/uwsgi.ini
  .. code-block::

   [uwsgi]
   chdir = /var/www/ocr_server
   module = ocr_server.wsgi
   home = /var/www/ocr_server/venv
   master = true
   processes = 10
   http = 127.0.0.1:8003
   vacuum = true

 Create the systemd service unit /etc/systemd/system/django-ocr-server.service
   .. code-block::

    [Unit]
    Description=uWSGI Django OCR Server
    After=syslog.service

    [Service]
    User=nginx
    Group=nginx
    Environment="PATH=/var/www/ocr_server/venv/bin:/sbin:/bin:/usr/sbin:/usr/bin"
    ExecStart=/var/www/ocr_server/venv/bin/uwsgi --ini /var/www/ocr_server/uwsgi.ini
    RuntimeDirectory=uwsgi
    Restart=always
    KillSignal=SIGQUIT
    Type=notify
    StandardError=syslog
    NotifyAccess=all

    [Install]
    WantedBy=multi-user.target

 Reload systemd service
  $sudo systemctl daemon-reload
 Chango user of /var/www/ocr_server to nginx
  $sudo chown -R nginx:nginx /var/www/ocr_server
 Start Django-ocr-server service
  $sudo systemctl start django-ocr-service
 Check that port is up
  $sudo netstat -anlpt \| grep 8003
   | you have to got something like this:
   | tcp        0      0 127.0.0.1:8003          0.0.0.0:*               LISTEN      2825/uwsgi
 Enable Django-ocr-server uwsgi service
  $sudo systemctl enable django-ocr-service

 Edit /etc/nginx/nginx.conf
  .. code-block::

   server {
       listen       80 default_server;
       listen       [::]:80 default_server;
       server_name  _;
       charset utf-8;
       client_max_body_size 75M;
       location /static/rest_framework_swagger {
           alias /var/www/ocr_server/venv/lib/python3.6/site-packages/rest_framework_swagger/static/rest_framework_swagger;
       }
       location /static/rest_framework {
           alias /var/www/ocr_server/venv/lib/python3.6/site-packages/rest_framework/static/rest_framework;
       }
       location /static/admin {
           alias /var/www/ocr_server/venv/lib/python3.6/site-packages/django/contrib/admin/static/admin;
       }
       location / {
           proxy_pass http://127.0.0.1:8003;
       }
   }

 Configure selinux
  .. code-block::

   $sudo semanage port -a -t http_port_t -p tcp 8003
   $sudo semanage fcontext -a -t httpd_sys_content_t '/var/www/ocr_server/venv/lib/python3.6/site-packages/rest_framework_swagger/static/rest_framework_swagger(/.*)?'
   $sudo restorecon -Rv '/var/www/ocr_server/venv/lib/python3.6/site-packages/rest_framework_swagger/static/rest_framework_swagger/'
   $sudo semanage fcontext -a -t httpd_sys_content_t '/var/www/ocr_server/venv/lib/python3.6/site-packages/rest_framework/static/rest_framework(/.*)?'
   $sudo restorecon -Rv '/var/www/ocr_server/venv/lib/python3.6/site-packages/rest_framework/static/rest_framework/'
   $sudo semanage fcontext -a -t httpd_sys_content_t '/var/www/ocr_server/venv/lib/python3.6/site-packages/django/contrib/admin/static/admin(/.*)?'
   $sudo restorecon -Rv '/var/www/ocr_server/venv/lib/python3.6/site-packages/django/contrib/admin/static/admin/'

 Start nginx service
  $sudo systemctl start nginx
 Enable nginx service
  $sudo systemctl enable nginx
 Configure firewall
  | $sudo firewall-cmd --zone=public --add-service=http --permanent
  | $sudo firewall-cmd --reload
 Go to http://{your_server}:80
   You will be redirected to admin page