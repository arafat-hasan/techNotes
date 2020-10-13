---
title: LAMP Installation and Config
date: 2020-09-25
category: ["professional writings"]
tag: ["note", "lamp", "installation", "centos"]
draft: false
type: posts
---


## Install phpMyAdmin on Centos 8

### Step 1: Install phpMyAdmin

- The phpMyAdmin tool is not included in the default CentOS 8 repositories. However, the file can be download manually.
```
wget https://files.phpmyadmin.net/phpMyAdmin/4.9.4/phpMyAdmin-4.9.4-all-languages.zip
```

- Extract the .zip file archive:
```
unzip phpMyAdmin-4.9.4-all-languages.zip
```

- Move the extracted files to the `/usr/share/phpmyadmin` directory:
```
sudo mv phpMyAdmin-4.9.4-all-languages.zip /usr/share/phpmyadmin
```

- Change directories:
```
cd /usr/share/phpmyadmin
```

- Rename the sample php configuration file:
```
sudo mv config.sample.inc.php config.inc.php
```

- Open the php configuration file for editing:	
```
sudo nano config.inc.php
```

- Find the following line:
```
$cfg['blowfish_secret'] = '';
```

- Edit the line and enter the new php root password between the single-quotes, as follows:
```
$cfg['blowfish_secret']='my_password';
```


- Next, create and set permissions on a temporary phpMyAdmin directory:
```
mkdir /usr/share/phpmyadmin/tmp

chown -R apache:apache /usr/share/phpmyadmin

chmod 777 /usr/share/phpmyadmin/tmp
```

### Step 2: Configure Apache for phpMyAdmin

- Create an Apache configuration file:
```
sudo nano /etc/httpd/conf.d/phpmyadmin.conf
```

- This creates a new, blank configuration file. Enter the following code:

```
Alias /phpmyadmin /usr/share/phpmyadmin

<Directory /usr/share/phpmyadmin/>
   AddDefaultCharset UTF-8

   <IfModule mod_authz_core.c>
     # Apache 2.4  
     <RequireAny>
      Require all granted
     </RequireAny>
    </IfModule>
    <IfModule !mod_authz_core.c>
      # Apache 2.2
      Order Deny,Allow
      Deny from All
      Allow from 127.0.0.1
      Allow from ::1
    </IfModule>
</Directory>
   
<Directory /usr/share/phpmyadmin/setup/>
   <IfModule mod_authz_core.c>
     # Apache 2.4
     <RequireAny>
       Require all granted
     </RequireAny>
   </IfModule>
   <IfModule !mod_authz_core.c>
     # Apache 2.2
     Order Deny,Allow
     Deny from All
     Allow from 127.0.0.1
     Allow from ::1
   </IfModule>
</Directory>
```

- Save the file (Ctrl+o) and exit (Ctrl+x).

- Finally, restart the Apache service to apply the changes made in the configuration file:
```
systemctl restart httpd

```

### Step 3: Configure SELinux and Firewall

SELinux stands for Security-Enhanced Linux. This is a kernel-level enhancement that improves security. Reconfigure this protocol for phpMyAdmin to work.

- Start by installing the following software package:
```
yum -y install policycoreutils-python-utils
```

Some versions of CentOS 8 may already have this package installed. In that case, the output indicates it has nothing to do and you can move on to the next step.



- Next, enable access to the phpmyadmin directory with the following commands:
```
semanage fcontext -a -t httpd_sys_rw_content_t '/usr/share/phpmyadmin/'

semanage fcontext -a -t httpd_sys_rw_content_t "usr/share/phpmyadmin/tmp(/.*)?"

restorecon -Rv '/usr/share/phpmyadmin/'
```

The first two commands may take a moment to complete. The third command recurses through the phpmyadmin directory to apply the changes.



#### Adjust the Firewall to Allow Traffic

- Create a firewall rule to allow HTTP traffic with the command:
```
firewall-cmd --permanent --add-service=http
```
- Make sure to reload the firewall after making these modifications:
```
firewall-cmd --reload
```


### Step 4: Test phpMyAdmin

- Open a web browser, and navigate to the following URL:
```
localhost/phpmyadmin
```
The browser should display the phpMyAdmin login page. However, if you attempt to log in, an error message may appear:
```
"The server requested an authentication method unknown to the client."
```
This error occurs because MySQL 8.x upgraded the password authentication mechanism. PhpMyAdmin has not been updated yet to use this authentication method.

- To bypass this measure, open the MySQL shell and alter the root user:
```
mysql -u root -p

password

ALTER USER 'root'@'localhost' IDENTIFIED WITH myswl_native_password BY 'password';
```
- Replace password with the actual password you set when securing the MySQL installation.

- Refresh the web browser phpMyAdmin page, and log in with your MySQL username and password.


### Step 5: Restrict Unauthorized Access to phpMyAdmin (Optional)

You should now have a working phpMyAdmin utility. This section will help you prevent unauthorized access to sensitive databases.
Allow phpMyAdmin Only From a Specific IP Address

- Open the phpmyadmin.conf file in a text editor (we will be using nano):
```
sudo nano /etc/httpd/conf.d/phpmyadmin.conf
```
- Find the following sections:

Require all granted

- Replace these lines with the following:
```
Require ip your_system's_ip_address
Require ip ::1
```
- Save and close the file.



### If you are using SELinux, check this out

I was running CentOS 8 with SELinux enforcing, and I was getting an error (`mysqli_real_connect(): (HY000/2002): No such file or directory`) despite having all the configurations fixed correctly. I later got out of trouble after allowing MySQL connections through SELinux.

Check SELinux status using this command:
```bash
sestatus
```

Allow Apache to connect database through SELinux
```
setsebool httpd_can_network_connect_db 1
```

Use `-P` option makes the change permanent. Without this option, the boolean would be reset to 0 at reboot.
```
setsebool -P httpd_can_network_connect_db 1
```
 
### Secure phpMyAdmin
At this point, the phpMyAdmin instance is functioning properly. However, securing your phpMyAdmin instance from the outside world it an important task for you. In this section, we will show you how to secure your phpMyAdmin instance.

#### Allow phpMyAdmin from Specific IP

First, you will need to configure your phpMyAdmin to accessible only from your home connection's IP address.

You can configure it by editing `/etc/httpd/conf.d/phpmyadmin.conf` file:
```
nano /etc/httpd/conf.d/phpmyadmin.conf
```

Find the following lines:
```
     <RequireAny>
       Require all granted
     </RequireAny>
```

And, replace them with the following:
```
<RequireAny>
    Require ip your-home--connection-ip-address
    Require ip ::1
</RequireAny>
```

Save and close the file when you are finished.


#### Configure Extra Layer of Authentication

It is a good idea to password protect your phpmyadmin directory by setting up a basic authentication.

To do so, create a new authentication file using the htpasswd tool as shown below:
```
mkdir /etc/phpmyadmin
 htpasswd -c /etc/phpmyadmin/.htpasswd admin
```

You will be asked to provide your admin password as shown below:
```
New password: 
Re-type new password: 
Adding password for user admin
```

Next, you will need to configure Apache to use the .htpasswd file. You can do this by editing the file /etc/httpd/conf.d/phpmyadmin.conf.
```
nano /etc/httpd/conf.d/phpmyadmin.conf
```

Add the following lines below the line "AddDefaultCharset UTF-8":
```
    Options  +FollowSymLinks +Multiviews +Indexes
    AllowOverride None
    AuthType basic
    AuthName "Authentication Required"
    AuthUserFile /etc/phpmyadmin/.htpasswd
    Require valid-user
```

Save the file and restart the Apache service for changes to take effect:
```
systemctl restart httpd
```

### Access phpMyAdmin
Now, your phpMyAdmin instance is secured with an extra layer of security. Open your web browser and type the URL http://your-server-ip/phpmyadmin. You will be asked to enter the login credentials of the user you previously created.

Provide your admin username and password, then click on the OK button. You will be redirected to the phpMyAdmin login page. Now, provide your MySQL administrative user login credentials and click on the Go button.





স্নাপ সমস্যা:
এসই লিনাক্সে স্নাপ সমস্যা করছিলো, কিছু ক্ষণ পরপর এলার্ট করে।
এই লিংকে সমাধান আছে বলে মনে হলো:
https://forum.snapcraft.io/t/centos-8-snapd-installation-selinux/14998/4

The solution is to enable the continuous release (CR) repository 41 since it contains the upcoming 8.1 packages.

Enable CR repo:
> The repository configuration file is included in the newest centos-release package. First update your system with dnf update to get the new centos-release package, then run dnf config-manager --enable cr to enable the CR repository. 


SELinux Alert Browser এ এই সাজেশন দিয়েছে: তা চালালাম:
```
You should report this as a bug.
You can generate a local policy module to allow this access.
Allow this access for now by executing:
# ausearch -c 'snapd' --raw | audit2allow -M my-snapd
# semodule -X 300 -i my-snapd.pp
```



