# renew

A script, driven by vhost files, to have certbot create SSL certificates, or to renew those coming up for expiry.

## Example usage

```
/usr/local/bin/renew
```

## Setup

Merge `usr/` and `var/` to the root of your host machine.

If not already so, arrange your virtualhosts in a structure like this:

```
/path/to/vhosts/
    sites-available/
        www.example.com
        blog.example.com
    sites-enabled/
        www.example.com -> ../sites-available/www.example.com
        blog.example.com -> ../sites-available/blog.example.com
```

If there is a suffix to the files (e.g., `.conf`), you can specify it with the `-s` flag when calling renew.

In each vhost, configure renewal using these options:
 - domain: domain(s) for the cerificate, first specified is the primary
 - email: email address to pass to certbot for reminders etc.
 - post_renew: command(s) to run after renewing

Each option is specified in a comment preceeded by renew_conf. Here's an Nginx example, for a website which responds to three domains, and exports the resulting certificates to a mail server config as well:

```
# renew_conf domain www.example.com
# renew_conf domain example.com examp.le
# renew_conf email info@example.com
# renew_conf post_renew /bin/cat /etc/letsencrypt/live/www.example.com/{privkey,fullchain}.pem  > /etc/courier/imapd.pem
# renew_conf post_renew /usr/sbin/service courier-imap restart
# renew_conf post_renew /usr/sbin/service courier-imap-ssl restart
# renew_conf post_renew /usr/sbin/service courier-authdaemon restart

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;

    server_name www.example.com example.com examp.le;

    ssl_certificate /etc/letsencrypt/live/www.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.example.com/privkey.pem;

    root /var/www/html;
    index index.html;

    location / {
        try_files \$uri \$uri/ =404;
    }
}
```

Note that the actual vhost is not parsed in any way so you are responsible for referencing the certificate and setting the server name. Virtualhost errors or non-existent certs will not affect renewal.

For a simple website, you won't need to use the post_renew hook.

Now, run `renew` once a day or so, and any missing or soon-to-expire certificates will be renewed. Best to run it as a cron job.

Certificates for disabled sites are still renewed, but put back to disabled after renewing.

Unless verbose mode is enabled, the command is silent when no certificates are up for renewal. To suppress output even when certificate(s) are up for renewal, use the `-s` flag.

Options:
```
-a
    Use Apache mode.

-d
    Dry run: skip actual certbot and post_renew hook execution, and never restart the web server.

-e EMAIL
    Fallback email address for any vhost that does not specify one in its config.

-c PATH
    Config path: Path to the root of your webserver config.
    Default: `/etc/nginx` in Nginx mode, `/etc/apache2` in Apache mode.

-f
    Force: For existing certs, don't check if expiring soon; just proceed as if they are.

-n
    Use Nginx mode.
    This is the default mode.

-o VHOST
    Only one: only operate on the nominated vhost (if it exists).

-p
    Period: number of seconds within which certificate must be expiring before we may attempt to refresh.
    Default: 2160000 (equals 25 days).

-r
    Replace: pass the `--expand` flag to certbot. Only works with `-o`.

-s
    Silent: Suppress all output except errors.

-v
    Verbose: Explain what's happening as we go.

-x SUFFIX
    Suffix: config file suffix, e.g., `.conf` for apache under Ubuntu.
    Default: no suffix.
```

A good way to test your setup is with `renew -vfd`.
