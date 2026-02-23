# 1. Partitioning

## 1.1 physical volume disk layout

| disk | partition | type              | luks  | lvm   | label    | size      | format | mount                      |
| ---- | --------- | ----------------- | ----- | ----- | -------- | --------- | ------ | -------------------------- |
| 0    | 1         | efi               | false | false | boot     | 320M      | fat 32 | /boot                      |
| 0    | 2         | linux server data | true  | false | keys     | 256M      | luks   | none                       |
| 0    | 3         | linux file system | true  | true  | proc     | 23G       | luks   |  |
| 0    | 4         | linux server data | true  | true  | data     | 100% Free | luks   |  |

## 1.2 logical volume

### 1.2.1 disk layout proc
| partition | list | group  | name | size | mount                 | format |
| --------- | ---- | ------ | ---- | ---- | --------------------- | ------ |
| 3         | 1    | proc   | root | 15G  | /mnt                  | ext4   |
| 3         | 2    | proc   | vars | 3G   | /mnt/var              | ext4   |
| 3         | 3    | proc   | vlog | 512M | /mnt/var/log/         | ext4   |
| 3         | 4    | proc   | vaud | 256M | /mnt/var/log/audit    | ext4   |
| 3         | 5    | proc   | vtmp | 256M | /mnt/var/tmp/         | ext4   |
| 3         | 6    | proc   | vpac | 2.5G | /mnt/var/cache/pacman | ext4   |
| 3         | 7    | proc   | temp | 1.5G | /mnt/tmp              | ext4   |

### 1.2.2 disk layout data
| partition | list | group  | name | size     | mount                  | format |
| --------- | ---- | ------ | ---- | -------- | ---------------------  | ------ |
| 4         | 1    | data   | post | 100G     | /mnt/var/lib/postgres  | ext4   |
| 4         | 2    | data   | home | 100%free | /mnt/home              | ext4   |


# 2. preparation

## 2.1 instalation package
```
sudo pacman -S valkey postgresql gitlab nginx
```
# 3 configuration
## 3.1 generate random strings secrets
```
sudo hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab/secret
```
```
sudo chmod 640 /etc/webapps/gitlab/secret
```
```
sudo hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab-shell/secret
```
```
sudo chmod 640 /etc/webapps/gitlab-shell/secret
```
> After generate make sure that the files /etc/webapps/gitlab/secret and /etc/webapps/gitlab-shell/secret files contain strings secret. `cat /etc/webapps/gitlab/secret and /etc/webapps/gitlab-shell/secret`

## 3.2 valkey cache database

```
sudo systemctl enable valkey
```
```
sudo systemctl start valkey
```
```
sudo usermod -aG valkey gitlab
```
```
sudo nvim /etc/webapps/gitlab/cable.yml
```
Fill in the file

```
development:
  adapter: redis
  url: unix:/run/valkey/valkey.sock
  channel_prefix: gitlab_development
test:
  adapter: redis
  url: unix:/run/valkey/valkey.sock
  channel_prefix: gitlab_test
production:
  adapter: redis
  url: unix:/run/valkey/valkey.sock
  channel_prefix: gitlab_production
```
```
sudo nvim /etc/webapps/gitlab/resque.yml
```
Make sure url section development,test,production like this

```
development:
  url: unix:/run/valkey/valkey.sock
test:
  url: unix:/run/valkey/valkey.sock
production:
  url: unix:/run/valkey/valkey.sock
```
```
touch /run/valkey/valkey.sock
```
```
sudo chown -R valkey:valkey /run/valkey/valkey.sock
```

Add `unixsocket /run/valkey/valkey.sock` and `unixsocketperm 777` in `/etc/valkey/valkey.conf`

```
sudo nvim /etc/valkey/valkey.conf
```

uncommenting and edit
```
...

unixsocket /run/valkey/valkey.conf
unixsocketperm 777
```

## 3.3 postgresql database

Login to PostgreSQL and create the `gitlabhq_production` database along with its user.

```
sudo su - postgres
```
```
initdb -D /var/lib/postgres/data
```
```
exit
```
```
sudo systemctl enable postgresql
```
```
sudo systemctl start postgresql
```
```
sudo su - postgres
```
```
psql -d template1
```
> Remember to change your_username_here and your_password_here to the real values :
```
template1=# CREATE USER your_username_here WITH PASSWORD 'your_password_here';
template1=# ALTER USER your_username_here SUPERUSER;
template1=# CREATE DATABASE gitlabhq_production OWNER your_username_here;
template1=# \q
```

Try connecting to the new database with the new user to verify it works:
```
psql -d gitlabhq_production -U your_username_here -W
```
Open the new `/etc/webapps/gitlab/database.yml` and set the values for `username:` and `password:`. For example:
```
sudo nvim /etc/webapps/gitlab/database.yml
```
```
# PRODUCTION
#
production:
  main:
    adapter: postgresql
    encoding: unicode
    database: gitlabhq_production
    username: your_username_here
    password: "your_password_here"
    # host: localhost
    # port: 5432
    socket: /run/postgresql/.s.PGSQL.5432
```
add this section update main:

```
 ci:
    adapter: postgresql
    encoding: unicode
    database: gitlabhq_production
    database_tasks: false
    username: your_username_here
    password: "your_password_here"
    # host: localhost
    # port: 5432
    socket: /run/postgresql/.s.PGSQL.5432
```
> We only need to set up the production database to get GitLab working.

## 3.4 Initialize Gitlab database
```
sudo systemctl enable gitlab-gitaly.service
```
```
sudo systemctl start gitlab-gitaly.service
```
Initialize the database and activate advanced features:

```
sudo cd /usr/share/webapps/gitlab
```
```
sudo -u gitlab $(cat environment | xargs) bundle exec rake gitlab:setup DISABLE_DATABASE_ENVIRONMENT_CHECK=1
```
## 3.5 Nginx web server

```
sudo mkdir /etc/nginx/sites-available /etc/nginx/sites-enabled
```
```
nvim /etc/nginx/sites-available/gitlab
```
Fill in the file. For the example : 

```
upstream gitlab-workhorse {
  server unix:/run/gitlab/gitlab-workhorse.socket fail_timeout=0;
}

server {
  listen 80;                  # IPv4 HTTP
  #listen 443 ssl http2;      # uncomment to enable IPv4 HTTPS + HTTP/2
  #listen [::]:80;            # uncomment to enable IPv6 HTTP
  #listen [::]:443 ssl http2; # uncomment to enable IPv6 HTTPS + HTTP/2
  #server_name example.com;

  access_log  /var/log/gitlab/nginx_access.log;
  error_log   /var/log/gitlab/nginx_error.log;

  #ssl_certificate ssl/example.com.crt;
  #ssl_certificate_key ssl/example.com.key;

  location ~ ^/(assets)/ {
    root /usr/share/webapps/gitlab/public;
    gzip_static on; # to serve pre-gzipped version
    expires max;
    add_header Cache-Control public;
  }

  location / {
      # unlimited upload size in nginx (so the setting in GitLab applies)
      client_max_body_size 0;

      # proxy timeout should match the timeout value set in /etc/webapps/gitlab/puma.rb
      proxy_read_timeout 60;
      proxy_connect_timeout 60;
      proxy_redirect off;

      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;

      #proxy_set_header X-Forwarded-Ssl on;

      proxy_pass http://gitlab-workhorse;
  }

  error_page 404 /404.html;
  error_page 422 /422.html;
  error_page 500 /500.html;
  error_page 502 /502.html;
  error_page 503 /503.html;
  location ~ ^/(404|422|500|502|503)\.html$ {
    root /usr/share/webapps/gitlab/public;
    internal;
  }
}
```
```
sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab
```
Append include `sites-enabled/*;` to the end of the `http block`:

```
nvim /etc/nginx/nginx.conf
```
```
http {
    ...
    include sites-enabled/*;
}
```
```
sudo systemctl restart nginx
```
# 4. Testing
## 4.1 start gitlab

```
sudo systemctl enable gitlab.target
```
```
sudo systemctl start gitlab.target
```
## 4.2. search browser

```
http://localhost:port
```
or

```
http://ip_Address:port
```
