# Installing Logbook Server Using Ansible

Ada beberapa langkah dalam membuat Logbook Server menggunakan Ansible

## Inisialisaikan dahulu untuk penamaan dan hosts ansible

```
  name: Configure a server for Laravel
  hosts: webserver
  become: yes
```

## Membuat variable penampung untuk nantinya dipakai pada saat dipanggil di fungsi.

Ansible akan meminta value, seperti nama domain, git clone, token cloudflare dan nama database, seperti perintah di bawah ini:

```
vars_prompt:
    - name: domain_name
      prompt: 'Silahkan masukkan nama Domain'
      private: no
    - name: github_name
      prompt: 'Masukkan repo github untuk di clone'
      private: no
    - name: token
      prompt: 'Please Input your Token'
      private: no
    - name: db_name
      prompt: 'Please insert db name'
      private: no
```

## Pengecekan Nama domain dan Pembuatan folder

Setelah memasukkan value nama domain pada inputan. Selanjutnya lakukan pengecekan nama domain,
apabila value telah dimasukkan akan menjalankan perintah membuat folder baru.

##### 1. Lakukan pengecekan, jika tidak memasukkan value akan muncul error tidak menginput nama Domain

```
 - name: Check if domain name is provided
      fail:
        msg: 'Tidak menginput nama Domain'
      when: domain_name == ''
```

##### 2. Menampilkan nama domain dan membuat folder baru berdasarkan nama domain yang dimasukkan

```
- name: Display the domain name
      debug:
        msg: 'Nama Domain anda adalah {{ domain_name }}'

    - name: Create domain directory
      file:
        path: '/var/www/{{ domain_name }}/'
        state: directory
        owner: root
        group: root
        mode: '0755'
```

## Update dan upgrade packages

Lakukan upgrade dan update packages untuk mendapatkan package terbaru dari OS

```
- name: Update and upgrade packages
      apt:
        update_cache: yes
        upgrade: dist
```

## Menginstall package yang dibutuhkan

Ada beberapa packages yang dibutuhkan untuk Logbook Server ini, di antaranya :

##### 1. Menginstall Tools dan dependency

Beberapa tool dependency yang dibutuhkan seperti curl, git dan termasuk untuk nginx serta certbot

```
- name: Install required packages
      apt:
        name:
          - curl
          - pwgen
          - git
          - vim
          - zip
          - unzip
          - software-properties-common
          - nginx
          - mysql-server
          - python3-certbot-dns-cloudflare
        state: present
```

##### 2. Menginstall pip dan PyMySQL

Karena menggunkan ansible pada membuat server, jadi perlu menginstall pip dan PyMySQL sebagai pengganti dari MySQL-Server

```
- name: Install pip
      apt:
        name: python3-pip
        state: present

    - name: Install PyMySQL
      pip:
        name: pymysql
        state: present
```

##### 3. Menambahkan Repository dan Menginstall PHP versi 7.4

Untuk kompatibel dengan versi laravel yang digunakan,
maka dari itu perlu ada perubahan versi PHP menggunakan versi terdahulu yaitu versi 7.4
Dan juga menambahkan beberapa extensions pada PHP seperti fpm, curl, dan lainnya

```
- name: Add PHP repository
      apt_repository:
        repo: ppa:ondrej/php
        state: present

    - name: Install PHP 7.4 and extensions
      apt:
        name:
          - php7.4-fpm
          - php7.4-gd
          - php7.4-mbstring
          - php7.4-iconv
          - php7.4-dom
          - php7.4-xml
          - php7.4-zip
          - php7.4-intl
          - php7.4-mysql
          - php7.4-mongodb
          - php7.4-pdo
          - php7.4-curl
          - php7.4-xmlrpc
          - php7.4-soap
        state: present
```

## Download dan Menjalankan Composer Installer

Mendownload composer untuk nantinya dijalankan saat akan konfigurasi laravel.

##### 1. Mendownload file composer

```
- name: Download and install Composer
      get_url:
        url: https://getcomposer.org/installer
        dest: /tmp/composer-setup.php
      register: composer_download
```

##### 2. Menjalankan composer installer

Perintah berikut untuk menjalankan composer installer dan menyimpan di folder /usr/local/bin,
Selain itu juga composer yang digunakan versi 2.4.1, hal ini untuk kompatible dengan versi laravel

```
- name: Run Composer installer
      command: php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer --version=2.4.1
```

## Clone Github Repository

Menjalankan perintah untuk clone github repository dari value inputan github_name.
Dan menyimpan hasil clone di folder /var/www/{{domain_name}}.
domain_name adalah nama domain yang sebelumnya diinputkan.

```
- name: Clone GitHub repository
      git:
        repo: '{{ github_name }}'
        dest: '/var/www/{{ domain_name }}/'
```

## Konfigurasi MySQL

Beberapa hal yang perlu disiapkan untuk konfigurasi MySQL, sebagai berikut:

##### 1. Create Database MySQL

Membuat database awal untuk nantinya akan disinkronkan dengan db pada laravel.
pada perintah berikut ada variable {{db_name}} itu artinya mengambil value nam database yang sebelumnya sudah diinputkan

```
- name: Create MySQL database
      mysql_db:
        login_user: root
        login_password: root
        login_unix_socket: /run/mysqld/mysqld.sock
        name: '{{ db_name }}'
        state: present
```

##### 2. Membuat User MySQL

Berikutnya membuat user untuk semua privileges database,
agar saat autentikasi menggunakan nama dan password yang sudah dibuat.

```
- name: Create MySQL user
      mysql_user:
        login_user: root
        login_password: root
        login_unix_socket: /run/mysqld/mysqld.sock
        name: noc
        password: star_net2016
        priv: '*.*:ALL'
        host: localhost
        state: present
```

##### 3. Import database dari github clone

Langkah terakhir, lakukan import ke database dari folder clone github yang tadi sudah dijalan kan.

```
- name: Import database
      mysql_db:
        login_user: root
        login_password: root
        login_unix_socket: /run/mysqld/mysqld.sock
        name: '{{ db_name }}'
        state: import
        target: '/var/www/{{ domain_name }}/{{ db_name }}.sql'
```

## Konfigurasi dan Menjalankan Certbot

Pada perintah berikut, akan menjalankan konfigurasi certbot dengan membuat file cloudflare.ini
pada folder /etc/letsencrypt/ dan memasukkan token yang cloudflare yang sudah diinputkan sebelumnya.
Setelah file dibuat, selanjutnya menjalankan perintah credentials untuk domain yang diinputkan

```
- name: Configure Certbot
      copy:
        dest: '/etc/letsencrypt/cloudflare.ini'
        content: |
          dns_cloudflare_api_token={{ token }}

- name: Run Certbot
      command: certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini -d {{ domain_name }}
```

## Konfigurasi Webserver nginx

Ada beberapa langkah untuk konfigurasi nginx, sebagai berikut:

##### 1. Membuat File .conf

Membuat file .conf dengan template nama domain yang telah diinputkan, lalu disimpan di folder /etc/nginx/sites-available/ ,
lalu memasukkan konfigurasi berikut:

```
- name: Configure Nginx
      copy:
        dest: '/etc/nginx/sites-available/{{ domain_name }}.conf'
        content: |
          server {
              server_name {{ domain_name }};

              add_header X-Frame-Options "SAMEORIGIN";
              add_header X-XSS-Protection "1; mode=block";
              add_header X-Content-Type-Options "nosniff";
              root /var/www/{{ domain_name }}/public;

              index index.php index.html index.htm index.nginx-debian.html;

              access_log /var/log/nginx/{{ domain_name }}.access.log;
              error_log /var/log/nginx/{{ domain_name }}.error.log error;

              location / {
                  try_files $uri $uri/ /index.php?$args;
              }

              location ~ \.php$ {
                  include snippets/fastcgi-php.conf;
                  fastcgi_pass unix:/run/php/php7.4-fpm.sock;
                  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
              }

              location ~ /\.ht {
                  deny all;
              }

              listen 443 ssl; # managed by Certbot
              ssl_certificate /etc/letsencrypt/live/{{ domain_name }}/fullchain.pem; # managed by Certbot
              ssl_certificate_key /etc/letsencrypt/live/{{ domain_name }}/privkey.pem; # managed by Certbot
              include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
              ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
          }
          server {
              if ($host = {{ domain_name }}) {
                  return 301 https://$host$request_uri;
              } # managed by Certbot

              listen 80;
              server_name {{ domain_name }};
              return 404; # managed by Certbot
          }

```

##### 2. Enable Konfigurasi Nginx

Setelah nama file dibuat, lakukan enable link ke folder /etc/nginx/sites-enabled/

```
- name: Enable Nginx configuration
      file:
        src: '/etc/nginx/sites-available/{{ domain_name }}.conf'
        dest: '/etc/nginx/sites-enabled/{{ domain_name }}.conf'
        state: link

```

##### 3. Restart Nginx Server

Langkah terakhir, lakukan restart pada Nginx server

```
    - name: Restart Nginx service
      service:
        name: nginx
        state: restarted

```

## Konfigurasi Environment File Laravel

Ada beberapa hal yang perlu dijalankan pada konfigurasi laravel ini, di antaranya:

##### 1. Membuat file .env

Membuat file .env dan menyimpannya pada folder projek laravel, berikut perintahnya:

```
- name: Configure Laravel environment file
      copy:
        dest: '/var/www/{{ domain_name }}/.env'
        content: |
          APP_NAME=Laravel
          APP_ENV=local
          APP_KEY=base64:62CdnfJh7ueuZzu6rHWpDCpk91uNEU5adTmuJjKnDcA=
          APP_DEBUG=true
          APP_URL=https://{{ domain_name }}

          LOG_CHANNEL=stack

          DB_CONNECTION=mysql
          DB_HOST=127.0.0.1
          DB_PORT=3306
          DB_DATABASE={{ db_name }}
          DB_USERNAME=noc
          DB_PASSWORD=star_net2016

          BROADCAST_DRIVER=log
          CACHE_DRIVER=file
          QUEUE_CONNECTION=sync
          SESSION_DRIVER=file
          SESSION_LIFETIME=120

          REDIS_HOST=127.0.0.1
          REDIS_PASSWORD=null
          REDIS_PORT=6379

          MAIL_MAILER=smtp
          MAIL_HOST=smtp.mailtrap.io
          MAIL_PORT=2525
          MAIL_USERNAME=null
          MAIL_PASSWORD=null
          MAIL_ENCRYPTION=null
          MAIL_FROM_ADDRESS=null
          MAIL_FROM_NAME="${APP_NAME}"

          AWS_ACCESS_KEY_ID=
          AWS_SECRET_ACCESS_KEY=
          AWS_DEFAULT_REGION=us-east-1
          AWS_BUCKET=

          PUSHER_APP_ID=
          PUSHER_APP_KEY=
          PUSHER_APP_SECRET=
          PUSHER_APP_CLUSTER=mt1

          MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
          MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```

##### 2. Menjalankan Composer Install dan Composer Dump

Menjalankan composer install dan dump pada folder projek laravel nya agar bisa dijalankan.

```
- name: Run Composer install
      become: yes
      command: composer install --no-interaction --optimize-autoloader
      args:
        chdir: '/var/www/{{ domain_name }}'

- name: Run Composer Dump
      become: yes
      command: composer dump-autoload --no-interaction --optimize
      args:
        chdir: '/var/www/{{ domain_name }}'
```

##### 3. Membuat PHP Keygen dan Menghapus Clear artisan

```
- name: Run PHP artisan keygen and Clear cache
      shell: php artisan key:generate && php artisan cache:clear
      args:
        chdir: '/var/www/{{ domain_name }}'
```

##### 4. Set Permissions

Setting Permissions pada folder laravel agar dapat diakses

```
- name: Set permissions for Laravel
      file:
        path: '/var/www/{{ domain_name }}/storage'
        state: directory
        owner: www-data
        group: www-data
        mode: '0775'

- name: Set permissions for Laravel bootstrap cache
      file:
        path: '/var/www/{{ domain_name }}/bootstrap/cache'
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
```

## Menjalankan Ansible Playbook

Langkah terakhir menjalankan ansible playbook pada terminal

```
ansible-playbook -i inventory.ini install-logbook.yml
```
