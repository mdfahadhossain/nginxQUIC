Install mySQL

```
sudo apt install mysql-server mysql-client
```

Start database
```
sudo systemctl start mysql
```

Enable mysql
```
sudo systemctl enable mysql
```

Database setup
```
sudo mysql_secure_installation
```

`yes` to all

Login to DB
```
sudo mysql -u root -p
```

Create Database
```
create database wordpress
```

create user for the database
```
create user "wpadmin"@"localhost" identified by "strongPASSword";
```

allow access
```
grant all privileges on wordpress.* to "wpadmin"@"localhost";
```

reload & exit
```
flush privileges;
exit;
```

## Install PHP
```
sudo apt install -y php7.4 php7.4-gd php7.4-mysql php7.4-zip php7.4-fpm
```

Download wordpress
```
wget https://wordpress.org/latest.zip
```

Extract wordpress
```
unzip latest.zip
```

> Install **unzip** if you don't have it installed.
> ```
> sudo apt install -y unzip
> ```

make neccessary directories
```
sudo mkdir -p /var/www
sudo mkdir -p /var/www/html
```

copy wordpress to `/var/www/html/`
```
sudo cp wordpress /var/www/html/
```

change ownership
```
sudo chown -R www-data:www-data /var/www
```

## Nginx config

configure the site
```
sudo nano sites-available/yoursite.com
```

paste the config
```
server {
        ## Your website name goes here.
        server_name yoursite.com;
        ## Your only path reference.
        root /var/www/html/wordpress;
        ## This should be in your http block and if it is, it's not needed here.
        index index.php index.html;

        location = /favicon.ico {
                log_not_found off;
                access_log off;
        }

        location = /robots.txt {
                allow all;
                log_not_found off;
                access_log off;
        }

        location / {
                # This is cool because no php is touched for static content.
                # include the "?$args" part so non-default permalinks doesn't break when using query string
                try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php$ {
                #NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
                include fastcgi_params;
                fastcgi_intercept_errors on;
                fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
                #The following parameter can be also included in fastcgi_params file
                fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
                expires max;
                log_not_found off;
        }
}
```

test nginx config
```
sudo nginx -t
```

restart nginx
```
sudo systemctl restart nginx
```

the site should go live
install ssl

```
sudo certbot --nginx -d yoursite.com
```

> if you don't have **certbot** installed
> ```
> sudo snap install certbot --classic
> ```

restart nginx once for the last
```
sudo systemctl restart nginx
```

Now, configure wordpress by visiting your site and enjoy.
