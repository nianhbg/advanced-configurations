# Securing SWAG
[SWAG](https://github.com/linuxserver/docker-swag) - Secure Web Application Gateway (formerly known as linuxserver/letsencrypt) is a full fledged web server and reverse proxy with Nginx, PHP7, Certbot (Let's Encrypt™ client) and Fail2Ban built in. SWAG allows you to expose applications to the internet, doing so comes with a risk and there are security measures that help reduce that risk. This article details how to configure SWAG and enhance it's security.

## Requirements

- A working instance of [SWAG](https://github.com/linuxserver/docker-swag)

## Monitor SWAG
Use monitoring solutions such as [SWAG Dashboard](https://github.com/linuxserver/docker-mods/tree/swag-dashboard) to keep an eye on the traffic going through SWAG and check for suspicious activity such as:

- A lot of hits from a country unrelated to your users
- A lot of requests to a specific page or static file
- Referers that shouldn't refer to your domain
- A lot of hits on status codes that are not 2xx

## Internal Applications
Internal applications can be proxied through SWAG in order to use `app.mydomain.com` instead of ip:port, and block them externally so only your local network could access them.

Create a file called `nginx/internal.conf` with the following configuration:

```Nginx
allow 192.168.1.0/24; #Replace with your LAN subnet
deny all;
```

Utilize the lan filter in your configuration by adding the following line inside every location block for every application you want to protect.
```
    include /config/nginx/internal.conf;
```

Example:

```Nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name collabora.*;
    include /config/nginx/ssl.conf;
    client_max_body_size 0;

    location / {
        include /config/nginx/internal.conf;
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app collabora;
        set $upstream_port 9980;
        set $upstream_proto https;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
    }
}
```

Repeat the process for all internal applications and for every location block.

One way to securely access internal applications from the internet is through a VPN, for example WireGuard:

[WireGuard Container](https://hub.docker.com/r/linuxserver/wireguard)

[WireGuard on OPNSense](https://blog.linuxserver.io/2019/11/16/setting-up-wireguard-on-opnsense-android/)


## Fail2Ban
Fail2Ban is an intrusion prevention software that protects external applications from brute-force attacks. Attackers that fail to login to your applications a certain number of times will get blocked from accessing all of your applications.
Fail2Ban looks for failed login attempts in log files, counts the failed attempts in a short period, and bans the IP address of the attacker.

Mount the application logs to SWAG's container by adding a volume for the log to the compose yaml:
```
      - /path/to/nextcloud/nextcloud.log:/nextcloud/nextcloud.log:ro
```
If the application has multiple log files with dates, mount the entire folder:
```
      - /path/to/jellyfin/log:/jellyfin:ro
```
Recreate the container with the log mount, then create a file called `nextcloud.local` under `fail2ban/filter.d`:
```nginx
[Definition]
failregex=^.*Login failed: '?.*'? \(Remote IP: '?<ADDR>'?\).*$
          ^.*\"remoteAddr\":\"<ADDR>\".*Trusted domain error.*$
ignoreregex =
```
The configuration file containes a pattern by which failed login attempts are matched. Test the pattern by failing to login to nextcloud and look for the entry corresponding to your failed attempt.
```
{"reqId":"k5j5H7K3eskXt3hCLSc4i","level":2,"time":"2020-10-14T22:56:14+00:00","remoteAddr":"1.2.3.4","user":"--",
"app":"no app in context","method":"POST","url":"/login","message":"Login failed: username (Remote IP: 5.5.5.5)",
"userAgent":"Mozilla/5.0 (Linux; Android 11; Pixel 5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/5.6.7.8 Mobile 
Safari/537.36","version":"19.0.4.2"}
```
Test the pattern in `nextcloud.local` by running the following command on the docker host:
```
docker exec swag fail2ban-regex /nextcloud/nextcloud.log /config/fail2ban/filter.d/nextcloud.local
```
If the pattern works, you will see matches corresponding to the amount of failed login attempts:
```
Lines: 92377 lines, 0 ignored, 2 matched, 92375 missed
[processed in 7.51 sec]
```
The final step is to activate the jail, add the following to `fail2ban/jail.local`:
```nginx
[nextcloud]
enabled = true
port    = http,https
filter  = nextcloud
logpath = /nextcloud/nextcloud.log
action  = iptables-allports[name=nextcloud]
```
The logpath is slightly different for applications that have multiple log files with dates:
```nginx
[jellyfin]
enabled  = true
filter   = jellyfin
port     = http,https
logpath  = /jellyfin/log*.log
action   =  iptables-allports[name=jellyfin]
```

Repeat the process for every external application, you can find Fail2Ban configurations for most applications on the internet.

If you need to unban an IP address that was blocked, run the following command on the docker host:
```
docker exec swag fail2ban-client unban <ip address>
```

This great mod sends a discord notification when Fail2Ban blocks an attack: [f2bdiscord](https://github.com/linuxserver/docker-mods/tree/swag-f2bdiscord).

## Geoblock
Geoblock reduces the attack surface of SWAG by restricting access based on countries.

Enable geoblock using either [DBIP mod](https://github.com/linuxserver/docker-mods/tree/swag-dbip) or [Maxmind mod](https://github.com/linuxserver/docker-mods/tree/swag-maxmind), follow the mod's instructions to set it up.

The mods come with 3 definitions for `$geo-whitelist`, `$geo-blacklist`, `$lan-ip`.

An example for allowing a single country:
```Nginx
map $geoip2_data_country_iso_code $geo-whitelist {
    default no;
    UK yes; #Replace with your country code list https://dev.maxmind.com/geoip/legacy/codes/iso3166/
}
```
An example for blocking high risk countries: (GilbN's list based on the Spamhaus statistics and Aakamai’s state of the internet report)
```Nginx
map $geoip2_data_country_iso_code $geo-blacklist {
    default yes; #If your country is listed below, remove it from the list
    CN no; #China
    RU no; #Russia
    HK no; #Hong Kong
    IN no; #India
    IR no; #Iran
    VN no; #Vietnam
    TR no; #Turkey
    EG no; #Egypt
    MX no; #Mexico
    JP no; #Japan
    KR no; #South Korea
    KP no; #North Korea
    PE no; #Peru
    BR no; #Brazil
    UA no; #Ukraine
    ID no; #Indonesia
    TH no; #Thailand
 }
```

Utilize the geoblock in your configuration by adding one of the following lines above your location section in every application you want to protect.

**Note that when using a whitelist filter, you also need to check if the source is a LAN IP, it's not required when using a blacklist filter.**
```nginx
    if ($lan-ip = yes) { set $geo-whitelist yes; }
    if ($geo-whitelist = no) { return 404; }
```
Or
```nginx
    if ($geo-blacklist = no) { return 404; }
```

Example:

```Nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name authelia.*;
    include /config/nginx/ssl.conf;
    client_max_body_size 0;

    if ($lan-ip = yes) { set $geo-whitelist yes; } #Check for a LAN IP
    if ($geo-whitelist = no) { return 404; } #Check the country filter

    location / {
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app authelia;
        set $upstream_port 9091;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
    }
}
```

Add the lines to every external application based on your needs.


## NGINX Configuration
### Google FLoC
Google will track any user visiting your website even if it doesn't have Google analytics or any other services related to Google. One easy way for users visiting websites to opt out of this is to not use Google Chrome and use browsers like Firefox, etc. However, website maintainers can also help against this new tracking technology by opting out of the FLoC network.

Add the following config line to `ssl.conf` to opt out:
```
add_header Permissions-Policy "interest-cohort=()";
```

### X-Robots-Tag
You can prevent applications from appearing in results of search engines and web crawlers, regardless of whether other sites link to it. It doesn't work on all search engines and web crawlers, but it significantly reduces the amount.

Add the X-Robots-Tag config line to `ssl.conf` to enable it on **all** of your applications:
```
add_header X-Robots-Tag "noindex, nofollow, nosnippet, noarchive";
```

Disable on a specific application and allow search engines to display it by add the following line to the application config inside the server tag:
```
add_header X-Robots-Tag "";
```

### HSTS
HTTP Strict Transport Security (HSTS) is a web security policy mechanism that helps to protect websites against man-in-the-middle attacks such as protocol downgrade attacks and cookie hijacking. It allows web servers to declare that web browsers (or other complying user agents) should automatically interact with it using only HTTPS connections, which provide Transport Layer Security (TLS/SSL), unlike the insecure HTTP used alone.

**HSTS requires a working SSL certificate on your domains before enabling it.**

Enable HSTS by uncommenting the HSTS config line in ssl.conf:
```
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

#### Optional - Strengthening HSTS
After enabling the HSTS header, users are still vulnerable to attack if they access an HSTS‑protected website over HTTP when they have:

- Never before visited the site
- Recently reinstalled their operating system
- Recently reinstalled their browser
- Switched to a new browser
- Switched to a new device (for example, mobile phone)
- Deleted their browser’s cache
- Not visited the site recently and the max-age time has passed

To address this, Google maintains a “HSTS preload list” of web domains and subdomains that use HSTS and have submitted their names to [HSTS Preload](https://hstspreload.org/). This domain list is distributed and hardcoded into major web browsers. Clients that access web domains in this list automatically use HTTPS and refuse to access the site using HTTP.

Be aware that once you set the STS header or submit your domains to the HSTS preload list, it is impossible to remove it. It’s a one‑way decision to make your domains available over HTTPS.


## Authelia
Authelia is an open-source authentication and authorization server providing 2-factor authentication and single sign-on (SSO) for your applications via a web portal. Refer to this [blog post to configure Authelia](https://blog.linuxserver.io/2020/08/26/setting-up-authelia/).


