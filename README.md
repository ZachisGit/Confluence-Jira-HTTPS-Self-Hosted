# Confluence-Jira-HTTPS-Self-Hosted

UPDATED VERSION OF THIS: https://gist.github.com/dborin/dd501b28967d3784fa646534dbea6ffa#file-jira_letsencrypt-md

HOWTO Configure Atlassian Confluence / Jira to use Letsencrypt certificate with default Tomcat
-----------------------------

This is a primer for installing a Letsencrypt certificate on a Jira server that is running the Confluence / Jira provided, default Tomcat for serving webpages.

I found lots of information about how to do it using a free-standing Tomcat or nginx, but nothing about this particular combination.  I hope it helps you!

Obviously, in all the examples, you need to replace `jira.example.com` or `confl.example.com` with your own domain!  And (duh) you need to use your own password, not `1234`

You need to have installed Java (`apt install openjdk-8-jre-headless`).  Then in your user's shell RC file and probably `root`'s RC file, add 

`export JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")`

**Jira / Confluence should NOT be running while you're doing this.**

Stopping Confluence/Jira:

    $ /etc/init.d/confluence stop
    $ /etc/init.d/jira stop

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

    $ sudo certbot certonly --standalone -d jira.example.com # Ubuntu
    $ sudo ./certbot-auto certonly --standalone -d jira.example.com # CentOS/RHEL

### Set it all up
------------------

I did this on an Ubuntu 16.04 machine.  I used the OpenJDK 8 for my Java install, so my `$JAVA_HOME` is `/usr/lib/jvm/java-8-openjdk-amd64/jre`

    $ sudo su - # Become root (much easier)
        
    # cd $JAVA_HOME

<b>Create a PKCS12 that contains both your full chain and the private key (IMPORTANT: leave the password empty)</b>

Note: -in and -inkey parameters are different

Confluence:

    # openssl pkcs12 -export -out /tmp/confl.example.com_fullchain_and_key.p12 -in /etc/letsencrypt/live/confl.example.com/fullchain.pem -inkey /etc/letsencrypt/live/confl.example.com/privkey.pem -name confl

Jira:

    # openssl pkcs12 -export -out /tmp/jira.example.com_fullchain_and_key.p12 -in /etc/letsencrypt/live/jira.example.com/fullchain.pem -inkey /etc/letsencrypt/live/jira.example.com/privkey.pem -name jira


<b>Convert that PKCS12 to a JKS</b>

Confluence:

    # keytool -importkeystore -deststorepass 1234 -destkeypass 1234 -destkeystore confl.jks -srckeystore /tmp/confl.example.com_fullchain_and_key.p12 -srcstoretype PKCS12 -srcstorepass "" -alias confl

Jira:

    # keytool -importkeystore -deststorepass 1234 -destkeypass 1234 -destkeystore jira.jks -srckeystore /tmp/jira.example.com_fullchain_and_key.p12 -srcstoretype PKCS12 -srcstorepass "" -alias jira

<b>If the system gives you a warning about PKCS12, it may tell you to run the following.  Go ahead.</b>

Confluence:

    # keytool -importkeystore -srckeystore confl.jks -destkeystore confl.jks -deststoretype pkcs12
    # keytool -list -alias confl -keystore $JAVA_HOME/confl.jks   # Check if it worked (not needed)
  
Jira:

    # keytool -importkeystore -srckeystore jira.jks -destkeystore jira.jks -deststoretype pkcs12
    # keytool -list -alias jira -keystore $JAVA_HOME/jira.jks   # Check if it worked (not needed)
    
 

### Edit Server Config - Confluence
`CONFLUENCE_INSTALL` by default is set to '/opt/atlassian/confluence/'

Create a backup of `<CONFLUENCE_INSTALL>/conf/server.xml` before editing it.  Edit the HTTP connector so that it redirects to the HTTPS connector (copy it exactly like this):

    <Connector acceptCount="100" connectionTimeout="20000" disableUploadTimeout="true" enableLookups="false" maxHttpHeaderSize="8192" maxThreads="150" minSpareThreads="25" port="8080" protocol="HTTP/1.1" redirectPort="8443" useBodyEncodingForURI="true"/>

Edit or <b>Create</b> the HTTPS connector so that it has the parameters that point to the KeyStore (copy it exactly like this):

    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
        maxHttpHeaderSize="8192" SSLEnabled="true"
        maxThreads="150" minSpareThreads="25"
        enableLookups="false" disableUploadTimeout="true"
        acceptCount="100" scheme="https" secure="true"
        sslEnabledProtocols="TLSv1.2,TLSv1.3"
        clientAuth="false" useBodyEncodingForURI="true"
        keyAlias="jira" keystoreFile="/usr/lib/jvm/java-8-openjdk-amd64/jre/confl.jks"
        keystorePass="1234" keystoreType="JKS"/>

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

<b>Last-Step:</b>

You have to set your base-url in confluence, do <b>"Step 4. Change your confluence base URL to HTTPS"</b> of the following tutorial:

https://confluence.atlassian.com/doc/running-confluence-over-ssl-or-https-161203.html


### Edit Server Config - Jira
`<JIRA_INSTALL>` by default is set to `/opt/atlassian/jira/`

Create a backup of `<JIRA_INSTALL>/conf/server.xml` before editing it. Edit the HTTP connector so that it redirects to the HTTPS connector (copy it exactly like this):

    <Connector acceptCount="100" connectionTimeout="20000" disableUploadTimeout="true" enableLookups="false" maxHttpHeaderSize="8192" maxThreads="150" minSpareThreads="25" port="8080" protocol="HTTP/1.1" redirectPort="8443" useBodyEncodingForURI="true"/>

  Edit or <b>Create</b> the HTTPS connector so that it has the parameters that point to the KeyStore (copy it exactly like this):

    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
        maxHttpHeaderSize="8192" SSLEnabled="true"
        maxThreads="150" minSpareThreads="25"
        enableLookups="false" disableUploadTimeout="true"
        acceptCount="100" scheme="https" secure="true"
        sslEnabledProtocols="TLSv1.2,TLSv1.3"
        clientAuth="false" useBodyEncodingForURI="true"
        keyAlias="jira" keystoreFile="/usr/lib/jvm/java-8-openjdk-amd64/jre/jira.jks"
        keystorePass="1234" keystoreType="JKS"/>

Save the changes to `server.xml`

If redirection to HTTPS will be used (this is recommended), edit the `<JIRA_INSTALL>/atlassian-jira/WEB-INF/web.xml` file and add the following section at the end of the file, before the closing `</web-app>`. In this example, all URLs except attachments are redirected from HTTP to HTTPS.

    <security-constraint>
	    <web-resource-collection>
		    <web-resource-name>all-except-attachments</web-resource-name>
		    <url-pattern>*.jsp</url-pattern>
		    <url-pattern>*.jspa</url-pattern>
		    <url-pattern>/browse/*</url-pattern>
		    <url-pattern>/issues/*</url-pattern>
	    </web-resource-collection>
	    <user-data-constraint>
		    <transport-guarantee>CONFIDENTIAL</transport-guarantee>
	    </user-data-constraint>
    </security-constraint>
Restart JIRA after you have saved your changes as follows:

    $ /etc/init.d/jira stop      # Only needed if you didn't stop jira in the beginning
    $ /etc/init.d/jira start

[How to check if jira is running](https://confluence.atlassian.com/jirakb/how-to-check-the-jira-application-is-running-794499415.html)(ps -ef | grep JIRA)

See https://confluence.atlassian.com/adminjiraserver/running-jira-applications-over-ssl-or-https-938847764.html#RunningJIRAapplicationsoverSSLorHTTPS-commandline for Troubleshooting tips

Make sure to setup a cronjob that runs every 89 days to update the Letsencrypt certificate.

    $ sudo certbot renew

You can try it out by doing:

    $ sudo certbot renew --dry-run
    
Letsencrypt will lock you out if you try to renew too many times in a short period of time, so use the `--dry-run` option when testing to see if it works!

### Firewall
On ubuntu don't forget to add the HTTPS port to your firewall

    $ ufw allow from any to any port 8443
