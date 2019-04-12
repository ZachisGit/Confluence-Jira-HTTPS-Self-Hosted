# Confluence-Jira-HTTPS-Self-Hosted

<i>References: </i><br>
    https://gist.github.com/dborin/dd501b28967d3784fa646534dbea6ffa#file-jira_letsencrypt-md
    https://confluence.atlassian.com/kb/securing-your-atlassian-applications-with-apache-using-ssl-838284349.html#

HOWTO Configure Atlassian Confluence to use Letsencrypt certificate with apache proxy
-----------------------------

This is a primer for installing a Letsencrypt certificate on a confluence server that is running the Confluence provided, default Tomcat for serving webpages togehter with an apache web server used here as a https proxy.

Obviously, in all the examples, you need to replace `confl.example.com` with your own domain!

**Confluence should NOT be running while you're doing this.**

Stopping Confluence:

    $ /etc/init.d/confluence stop

### Get Letsencrypt (certbot)
-----------------------------
For CentOS/RHEL

    $ wget https://dl.eff.org/certbot-auto
    $ chmod a+x certbot-auto

For Ubuntu (16.04)

    $ sudo apt-get update
    $ sudo apt-get install software-properties-common
    $ sudo add-apt-repository ppa:certbot/certbot
    $ sudo apt-get update
    $ sudo apt-get install certbot

### Get your certificate
-------------------------

    $ sudo certbot certonly --standalone -d confl.example.com # Ubuntu
    $ sudo ./certbot-auto certonly --standalone -d confl.example.com # CentOS/RHEL

### Set it all up
------------------

I did this on an Ubuntu 16.04 machine. 


### Edit Server Config - Confluence
`<CONFLUENCE_INSTALL>` by default is set to `/opt/atlassian/confluence/`

Create a backup of `<CONFLUENCE_INSTALL>/conf/server.xml` before editing it.  Edit the HTTP connector so that it redirects to the HTTPS connector (copy it exactly like this):

    <Connector acceptCount="100" connectionTimeout="20000" disableUploadTimeout="true" enableLookups="false" maxHttpHeaderSize="8192" maxThreads="150" minSpareThreads="25" port="8090" protocol="HTTP/1.1" redirectPort="8443" useBodyEncodingForURI="true"/>

Edit or <b>Create</b> the HTTPS connector so that it has the parameters that point to the KeyStore (copy it exactly like this):

    <Connector port="8090" connectionTimeout="20000" redirectPort="8443"
        maxThreads="200" minSpareThreads="10"
        enableLookups="false" acceptCount="10" debug="0" URIEncoding="UTF-8"
        proxyName="confl.example.com"
        proxyPort="443"
        secure="true"
        scheme="https" />


Save the changes to `server.xml`

If redirection to HTTPS will be used (this is recommended), edit the `<CONFLUENCE_INSTALL>/confluence/WEB-INF/web.xml` file and add the following section at the end of the file, before the closing `</web-app>`. In this example, all URLs except attachments are redirected from HTTP to HTTPS.

    <security-constraint>
      <web-resource-collection>
        <web-resource-name>Restricted URLs</web-resource-name>
          <url-pattern>/</url-pattern>
      </web-resource-collection>
      <user-data-constraint>
        <transport-guarantee>CONFIDENTIAL</transport-guarantee>
      </user-data-constraint>
    </security-constraint>

Restart CONFLUENCE after you have saved your changes as follows:

    $ /etc/init.d/confluence stop      # Only needed if you didn't stop confluence in the beginning
    $ /etc/init.d/confluence start

[How to check if confluence is running (article for jira but works the same for confluence)](https://confluence.atlassian.com/jirakb/how-to-check-the-jira-application-is-running-794499415.html)(ps -ef | grep CONFLUENCE)


### Edit Apache Config
`<APACHE_STANDARD_PATH>` by default is set to `/etc/apache2/sites-available/`

Make sure you delete all but the one file you configure from the ` /etc/apache2/sites-enabled` path.
In the one configuration file write the following:

    <VirtualHost *:443>
        ServerName confl.example.com

        ProxyRequests On

        <Proxy *>
             Require all granted
        </Proxy>

        ProxyPass / http://confl.example.com:8090/
        ProxyPassReverse / http://confl.example.com:8090/

        SSLEngine On
        SSLCertificateFile /etc/letsencrypt/live/confl.example.com/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/confl.example.com/privkey.pem

    </VirtualHost>

    <VirtualHost *:80>
        ServerName confl.example.com
        RewriteEngine On
        Redirect Permanent / https://confl.example.com/
    </VirtualHost>


<b>Last-Step:</b>

You have to set your base-url in confluence, do <b>"Step 4. Change your confluence base URL to HTTPS"</b> of the following tutorial:

https://confluence.atlassian.com/doc/running-confluence-over-ssl-or-https-161203.html



### Certificate Renewal
Make sure to setup a cronjob that runs every 89 days to update the Letsencrypt certificate.

    $ sudo certbot renew

You can try it out by doing:

    $ sudo certbot renew --dry-run
    
Letsencrypt will lock you out if you try to renew too many times in a short period of time, so use the `--dry-run` option when testing to see if it works!

### Firewall
On ubuntu don't forget to add the HTTPS port to your firewall

    $ ufw allow from any to any port 443
