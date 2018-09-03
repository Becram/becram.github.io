---
title:  "Intrusion Dectection system with Fail2ban"
last_modified_at: 2018-03-20T16:00:58-04:00
categories: 
  - Network&System
tags:
  - security

header:
  teaser: /assets/images/IDS/IDS.png
toc: true
toc_label: "Steps"
---

Install and configure fail2ban to stop some hacking/DOS (Denial of Service) attempts on Pustakalaya Server. As this site’s been hit by the occasional DOS from people with an axe to grind and too much time on their hands,fail2ban is lifesaver. Please follow the steps below:

## Step 1: Installation

1. Install fail2ban through the method of your choice.
2. Edit the file /etc/fail2ban/jail.local and add the following section: {if jail.local not present then create it}
```
   enabled = true
   port = http,https
   filter = http-get-dos
   logpath = /var/log/apache2/access.log
   # maxretry is how many GETs we can have in the findtime period before getting narky
   maxretry = 300
   # findtime is the time period in seconds in which we're counting "retries" (300 seconds = 5 mins)
   findtime = 300
   # bantime is how long we should drop incoming GET requests for a given IP for, in this case it's 5 minutes
   bantime = 300
```

3. Now we need to create the filter file, so create the file /etc/fail2ban/filters.d/http-get-dos.conf and place the following contents in it: 

```
 #
   # Author: http://www.go2linux.org
   #
   [Definition]

   # Option: failregex
   # Note: This regex will match any GET entry in your logs, so basically all valid and not valid entries are a match.
   # You should set up in the jail.conf file, the maxretry and findtime carefully in order to avoid false positives.

   failregex = ^<HOST> -.*"(GET|POST).*

   # Option: ignoreregex
   # Notes.: regex to ignore. If this regex matches, the line is ignored.
   # Values: TEXT
```

4. Now we just need to restart fail2ban for the new jail & filter to come into affect:

```
   /etc/init.d/fail2ban restart
```


## Step 2: Testing 

If you want to test if it’s really working, a nice way to do so is to use ab (Apache Benchmark – part of the apache2-utils package), like this:

```
ab -n 500 -c 10 http://your-web-site-dot-com/

```

This will kick off 500 page-loads in 10 concurrent connections against your site. When the ban kicks in the page-loads will stop (as incoming GET requests from your IP will be dropped), then when the bantime expires you’ll be able to access the site again. If you then take a look in your /var/log/fail2ban.log file you should see something like this:


```
2013-06-22 05:37:21,943 fail2ban.actions: WARNING [http-get-dos] Ban YOUR_IP_ADDRESS
2013-06-22 05:42:22,341 fail2ban.actions: WARNING [http-get-dos] Unban YOUR_IP_ADDRESS
```