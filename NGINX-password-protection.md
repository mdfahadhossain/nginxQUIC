# Password-Protected Setup for Nginx

This guide explains how to password-protect your website deployed using Ubuntu, and Nginx using HTTP basic authentication.

## Prerequisites:
- Ubuntu server
- Nginx server
- SSH access with sudo privileges.

## Steps:

### 1. Install Apache2 Utilities:

Apache2 Utilities package includes `htpasswd` utility that is used to create and manage password files.

Command to install it:

```shell
sudo apt-get install apache2-utils
```

### 2. Create Password File:

Next, create a password file which will store the user credentials.

Replace `user1` with your chosen username. You'll be prompted to enter and confirm your password for the provided username.

```bash
sudo htpasswd -c /etc/nginx/nextpasswd user1
```

### 3. Configure Nginx:

Configure Nginx to use the password file for HTTP basic authentication.

Open your Nginx configuration file. It might be located at `/etc/nginx/sites-available/default` or at `/etc/nginx/sites-available/domain.com`.

Include these lines inside the server block where your location is configured (/ in case of the main domain):

```nginx
location / {
    auth_basic "Administrator’s Area";
    auth_basic_user_file /etc/nginx/nextpasswd;
    ...
}
```

### 4. Restart Nginx:

After configuring, validate and then reload Nginx service:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

When you now try to access your website, you would be asked for the username and password. This ensures that your website is now password protected.
