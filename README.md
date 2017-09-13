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
