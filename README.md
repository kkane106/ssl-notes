# ssl-notes

* The following is a consolidated guide to configuring an EC2 server to run a public Apache server securely on port 443. ***NOTE:*** You will need a registered DNS to complete the following, you cannot configure HTTPS (SSL/TLS) with a bare IP address.

### EC2 Instance & Security Group Configuration

* Launch an EC2 instance, and configure it's security group with the following rules:

* Required:

<table>
<thead>
  <tr>
    <th>Name</th>
    <th>Type</th>
    <th>Port</th>
    <th>Select</th>
    <th>Rule</th>
  </tr>
</thead>
<tbody>
<tr>
  <td>SSH</td>
  <td>TCP</td>
  <td>22</td>
  <td>Custom</td>
  <td>0.0.0.0/0::/0 (the default)</td>
</tr>
<tr>
  <td>HTTP</td>
  <td>TCP</td>
  <td>80</td>
  <td>Anywhere</td>
  <td>0.0.0.0/0::/0 (the default)</td>
</tr>
<tr>
  <td>HTTPS</td>
  <td>TCP</td>
  <td>443</td>
  <td>Anywhere</td>
  <td>0.0.0.0/0::/0 (the default)</td>
</tr>
</tbody>
</table>

* Optional:

<table>
<thead>
  <tr>
    <th>Name</th>
    <th>Type</th>
    <th>Port</th>
    <th>Select</th>
    <th>Rule</th>
  </tr>
</thead>
<tbody>
<tr>
  <td>Custom</td>
  <td>TCP</td>
  <td>8080</td>
  <td>My IP</td>
  <td>\*.\*.\*.\*</td>
</tr>
<tr>
  <td>MySQL / Aurora</td>
  <td>TCP</td>
  <td>3306</td>
  <td>My IP</td>
  <td>\*.\*.\*.\*</td>
</tr>
</tbody>
</table>

* Download `.pem` key

* Launch instance

### Assign Your Instance an Elastic IP

1. Go to _EC2 > Running Instances > Network & Security : Elastic IPs_

2. Click _"Allocate new address" > "Allocate"_

3. Select the new Elastic IP, click _"Actions" > "Associate address"_ and point it at the server you configured in the previous step

### Point Registered DNS at Elastic IP

1. Log into your DNS manager, go to "Advanced Configuration" (or something similar) for your DNS.

* Set (replace \*.\*.\*.\*.\* with your Elastic IP):

```
A Record        @          *.*.*.*.*
A Record        www        *.*.*.*.*
```

2. The previous step will take some non zero amount of time to complete. Move onto the next step for the time being.

### Install and Configure Apache Web Server

1. First, update your server's package manager (`yum`):

```bash
sudo yum update -y
```

* ***Explanation:*** the `-y` flag will automatically enter 'yes' for all prompts requiring a 'y/n' answer. For more information on `yum`, check out this Red Hat Linux cheet sheet https://access.redhat.com/articles/yum-cheat-sheet


2. Now, install an Apache Web Server, in this case, we are going to use `httpd24`:

```bash
sudo yum install -y httpd24
```

3. Let's start the server, and make sure that it will always be running on server start:

```bash
sudo service httpd start
sudo chkconfig httpd on
```

* ***Explanation:*** here we are referring to the Apache server daemon (httpd)

### Create a Permission Group and Set File Permissions

1. Now we will add our `ec2-user` to an apache group:

```bash
sudo usermod -a -G apache ec2-user
```

* ***Explanation:*** this literally says: "with elevated permissions, modify a user by adding them to a supplementary group called 'apache', the user 'ec2-user'"

2. Now exit your secure shell (type `exit` and press return), and then re-ssh (`ssh -i your.pem ec2-user@*.*.*.*.*`) to your server.

3. Check the group assignments:

```bash
groups
```

* ***Explanation:*** this should list all of the groups the current user is a member of, you should see 'apache' listed.

4. Set the `ec2-user` and `apache` group as owners of the web root directories for the Apache server:

```bash
sudo chown -R ec2-user:apache /var/www
```

* ***Explanation:*** `chown` changes the owner and owning group of files, the `-R` flag performs the operation recursively. Here we are setting the `ec2-user` and `apache` group as the owners of all files and directories within `/var/www` which is the Apache server's web root.

5. Modify file and directory permissions on web root.

```bash
sudo chmod 2775 /var/www
```

* ***Explanation:*** `chmod` is used to change permission levels (specify what actions can be performed) on a specific file/directory. The permissions use an octal notation (4="read", 2="write", 1="execute", 0="no permission"). The octals are combined to set additional permissions (e.g. 6="read & write", 7="read & write & execute"). The position of the numbers corresponds to who has what permission, User|Group|Other, so `777` would mean anyone could perform any action, where as `400` only the authorized user could interact with it, and only read it at that. The 2 in front of `2775` is an additional flag that says that when the file is executed by a user it should be executed with the group id of that user. Here is a handy break down of `chmod` https://www.computerhope.com/unix/uchown.htm

6. Set all of the directories permissions within web root to be readable, writeable, and executable for our user and group:

```bash
find /var/www -type d -exec sudo chmod 2775 {} \\;
```

* ***Explanation:*** this finds (`find`) all of the directories (`-type d`) within web root (`/var/www`), and executes (`-exec`) a command to change their permissions. The `{}` at then end expands each file name, and the `\;` is an escaped semi-colon to terminate the command. Notice, that this makes the directories executable/readable/writeable to the user and the group

7. Set all of the files permissions within web root to be open to be readable and writeable (but not executable) for our user and group:

```bash
find /var/www -type f -exec sudo chmod 0664 {} \\;
```

* ***Explanation:*** same as above, except the `f` designates file, and the files aren't executable (thus we set the first flag to 0, we don't want them executed at all)

8. Restart your Apache server

```bash
sudo service httpd restart
```

### Check on DNS Assignment

1. Attempt to navigate to your DNS name. At this point your should see the default Apache test page. If you want to make sure it is pointing at the correct server, add an `index.html` with some content in the `/var/www/html` directory of the server and refresh your browser, the html page you created should now be visible.

### Add SSL/TLS support for Apache

1. Install `mod24_ssl` --> this adds a module to your apache server which adds support for SSL/TLS to your server.

```bash
sudo yum install -y mod24_ssl
```

2. Restart Apache

```bash
sudo service httpd restart
```

3. Confirm that SSL module is installed:

* From a local terminal, try to contact your server on port 443 (SSL/TLS) over https (where \*.\*.\*.\*.\* is your server's IP).

```bash
curl -i https://*.*.*.*.*:443
```

* This ***will not work and should produce*** something like the following:

```
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
```

### Download Certbot

1. Enable *Extra Packages for Enterprise Linux (EPEL)*:

```bash
sudo yum-config-manager --enable epel
```

2. Get "Certbot" (the script that is going to make it easy to setup SSL/TLS for us):

```bash
cd ~
wget https://dl.eff.org/certbot-auto
```

* ***Explanation:*** `wget` is literally Web Get, it requests and downloads resources over the internet.

3. Run `ls` to make sure you've obtained `certbot-auto`:

```bash
>> ls
certbot-auto
```

4. Make `certbot-auto` executable

```bash
chmod a+x certbot-auto
```
 * ***Explanation:*** `a` stands for "ALL", meaning everyone, `+x` means add executable permissions. This command modifies the existing permissions on `certbot-auto` to allow it to be executable by anyone.

### Run Certbot and Obtain a Let's Encrypt Certificate

1. Run the `certbot-auto` script in debug mode (Amazon linux requires that it be run in debug mode and will throw an error otherwise):

```bash
sudo ./certbot-auto --debug
```

2. This will begin a very long dialogue:

* When prompted **`Is this ok [y/d/N]`** type **`y`** and press *Enter*.

* When prompted **`Enter email address (used for urgent renewal and security notices),`** type your **email address** and press *Enter*.

* When prompted:

```bash
-------------------------------------------------------------------------------
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.1.1-August-1-2016.pdf. You must agree
in order to register with the ACME server at
https://acme-v01.api.letsencrypt.org/directory
-------------------------------------------------------------------------------
(A)gree/(C)ancel:
```

  * Type **`A`** and press *Enter*

* When prompted if you would like to join the mailing list, enter either **`Y`** *or* **`N`** and press *Enter*

* When prompted:

```bash
No names were found in your configuration files. Please enter in your domain
name(s) (comma and/or space separated)  (Enter 'c' to cancel):
```

  * enter the name of your DNS like `mysite.com` and press *Enter*

* When prompted:

```bash
Obtaining a new certificate
Performing the following challenges:
tls-sni-01 challenge for example.com
tls-sni-01 challenge for www.example.com

We were unable to find a vhost with a ServerName or Address of example.com.
Which virtual host would you like to choose?
(note: conf files with multiple vhosts are not yet supported)
-------------------------------------------------------------------------------
1: ssl.conf                       |                       | HTTPS | Enabled
-------------------------------------------------------------------------------
Press 1 [enter] to confirm the selection (press 'c' to cancel):
```

  * enter **`1`** and press *Enter*

* When prompted:

```bash
We were unable to find a vhost with a ServerName or Address of example.com.
Which virtual host would you like to choose?
(note: conf files with multiple vhosts are not yet supported)
-------------------------------------------------------------------------------
1: ssl.conf                       |                       | HTTPS | Enabled
-------------------------------------------------------------------------------
Press 1 [enter] to confirm the selection (press 'c' to cancel):
```
  * enter **`1`** and press *Enter*

* When prompted:

```bash
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
-------------------------------------------------------------------------------
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites...
-------------------------------------------------------------------------------
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

  * enter **`2`** and press *Enter*

3. Now you should see this success message:

```bash
Congratulations! You have successfully enabled https://example.com and
https://www.example.com

You should test your configuration at:
https://www.ssllabs.com/ssltest/analyze.html?d=example.com
https://www.ssllabs.com/ssltest/analyze.html?d=www.example.com
-------------------------------------------------------------------------------

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/example.com/fullchain.pem. Your cert will
   expire on 2017-07-19. To obtain a new or tweaked version of this
   certificate in the future, simply run certbot-auto again with the
   "certonly" option. To non-interactively renew *all* of your
   certificates, run "certbot-auto renew"
....
```

### Setup Certificate Auto-Renewal for your Server with a Cron Job

* A cron job is a task that will run at a specified date and time, typically at an interval.

1. Open `/etc/crontab` in vi

```bash
sudo vi /etc/crontab
```

2. Add the following cronjob to the file, below the example:

```bash
# Auto renew SSL Cert with Certbot (or at least try) twice per day
39 1,13 * * * root /home/ec2-user/certbot-auto renew --no-self-upgrade
```

* ***Explanation:*** the above cron job will run at 1:39am and 1:39pm every day, with root permissions, and execute the certbot-auto renew script (which will make certbot attempt to renew it's certificate). Don't worry, if your cert is not up for renewal (90 days), this will harmlessly bounce, if it is up for renewal, this will ensure that the least possible amount of time passes without you having a valid cert.

3. Restart your cron server and your Apacher server:

```bash
sudo service crond restart
sudo service httpd restart
```

4. Navigate to your DNS, (e.g. type `mysite.com` into chrome), you should see your html page and it should be over `https://`
