# Setting up Nexus for OpenShift mirror repository

This repository contains my notes on setting up Sonatype Nexus repository on Ubuntu 18.04 server in AWS EC2.

1. Login to EC2 instance:

        ```
        ssh -i sshkey ubuntu@<publicIP>
        ```

2. Install JDK 1.8

        ```
        sudo apt install openjdk-8-jre
        ```

3. Install jq

        ```
        sudo apt install jq
        ```

4. Download Nexus Repository OSS from [https://www.sonatype.com/products/repository-oss-download](https://www.sonatype.com/products/repository-oss-download)

5. Setup user nexus

    ```
    sudo adduser nexus
    ```
6. Expand nexus code

    ```
    sudo su -
    cd /opt
    tar -xf /home/ubuntu/Downloads/nexus-3.30.1-01.tar.gz
    ```

7. Prepare certificate using keytool (self-signed)

    ```
    keytool -keystore etc/ssl/keystore.jks -storepass password -genkeypair -keyalg RSA -validity 3650 -dname "CN=nexus" -ext "SAN=IP:10.0.19.246" -keysize 4096 -alias jetty
    keytool -exportcert -alias jetty -keystore etc/ssl/keystore.jks -storepass password -rfc -file jetty.pem
    ```

8. Add certificate to ubuntu ca trust

    ```
    cp jetty.pem /usr/local/share/ca-certificates/nexus/nexus.crt
    sudo update-ca-certificates
    ```

9. Create nexus systemd entry `/etc/systemd/system/nexus.service`:

    ```
    [Unit]
    Description=nexus service
    After=network.target

    [Service]
    Type=forking
    LimitNOFILE=65536
    ExecStart=/opt/nexus-3.30.1-01/bin/nexus start
    ExecStop=/opt/nexus-3.30.1-01/bin/nexus stop
    User=nexus
    Restart=on-abort
    TimeoutSec=600

    [Install]
    WantedBy=multi-user.target
    ```

7. Customize nexus configuration:

    ```
    cp /opt/nexus-3.30.1-01/etc/nexus-default.properties /opt/sonatype-work/nexus3/etc/nexus.properties
    ```
    - Edit `/opt/sonatype-work/nexus3/etc/nexus.properties`

        ```
        application-port-ssl=8443
        nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-https.xml,${jetty.etc}/jetty-requestlog.xml
        ssl.etc=${karaf.data}/etc/ssl
        ```
    - Edit `/opt/nexus-3.30.1-01/etc/jetty/jetty-https.xml`

        ```
        . . .
          <New id="sslContextFactory" class="org.eclipse.jetty.util.ssl.SslContextFactory$Server">
          <Set name="certAlias">jetty</Set>
          <Set name="KeyStorePath"><Property name="ssl.etc"/>/keystore.jks</Set>
          <Set name="KeyStorePassword">password</Set>
          <Set name="KeyManagerPassword">password</Set>
          <Set name="TrustStorePath"><Property name="ssl.etc"/>/keystore.jks</Set>
          <Set name="TrustStorePassword">password</Set>
          <Set name="EndpointIdentificationAlgorithm"></Set>
        . . .
        ```

8. Start nexus

    ```
    systemctl start nexus
    ```

9. Setup docker repository

10. Setup openshift cli:

    ```
    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.6.28/openshift-client-linux.tar.gz
    tar -xf openshift-client-linux.tar.gz
    sudo mv oc /usr/local/bin/oc
    ```

11. Setup pull-secret - create pull-secret.txt from the content in [https://cloud.redhat.com/openshift/install/pull-secret](https://cloud.redhat.com/openshift/install/pull-secret)

12. Run mirror:

    ```
    export OCP_RELEASE=<release_version> 
    export LOCAL_REGISTRY='<ipaddr>:8444' 
    export LOCAL_REPOSITORY='ocp46' 
    export PRODUCT_REPO='openshift-release-dev' 
    export LOCAL_SECRET_JSON='pull-secret.txt' 
    export RELEASE_NAME="ocp-release" 
    export ARCHITECTURE="x86_64"
    ```

    ```
    oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}
    ```