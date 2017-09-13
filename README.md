# ssl-notes

### First, setup an instance if you have not done so already...

#### https://github.com/SkillDistillery/SD11/tree/master/unit_2/week1/AWS-EC2-Apache

### Second, make sure the instance has an elastic IP

### Third, point an actual DNS at it (you can't do this with a bare IP)


### Fourth do this...

Install
Since it doesn't seem like your operating system has a packaged version of Certbot, you should use our certbot-auto script to get a copy:

```
wget https://dl.eff.org/certbot-auto
chmod a+x certbot-auto
```

```
 sudo ./certbot-auto --debug -v --server https://acme-v01.api.letsencrypt.org/directory certonly -d $YOUR_SITE_NAME
```

```
How would you like to authenticate with the ACME CA?
-------------------------------------------------------------------------------
1: Apache Web Server plugin - Beta (apache)
2: Spin up a temporary webserver (standalone)
3: Place files in webroot directory (webroot)
-------------------------------------------------------------------------------
Select the appropriate number [1-3] then [enter] (press 'c' to cancel): 3
```

```
Select the webroot for diydumbdumb.com:
-------------------------------------------------------------------------------
1: Enter a new webroot
-------------------------------------------------------------------------------
Press 1 [enter] to confirm the selection (press 'c' to cancel): 1
Input the webroot for diydumbdumb.com: (Enter 'c' to cancel): /var/www/html
```

### Setup Auto-Renew

1. Move `certbot-auto` to `/lib` so that you don't accidentally delete it.

```
sudo mv ~/certbot-auto /lib
```

2. use `crontab -e` to open vi so that you can make a new cron job

```
crontab -e
```

3. Press `i` to enter insert mode, and add the following cron job to run auto renew twice every day at 3am and 3pm:

```
0 3,15 * * * /lib/certbot-auto renew
```

4: Save and quit vi with `:wq`

