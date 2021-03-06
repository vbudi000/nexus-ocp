# Setting up Nexus for OpenShift mirror repository

This repository contains my notes on setting up Sonatype Nexus repository on Ubuntu 18.04 server in AWS EC2. (exposed ports: 22, 8081, 8082, 8443, 8444)

1. Login to EC2 instance:

        ```
        ssh -i sshkey ubuntu@<publicIP>
        ```

2. Install JDK 1.8

        ```
        sudo apt update
        sudo apt install -y openjdk-8-jre
        ```

3. Install jq

        ```
        sudo apt install -y jq
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
    cd sonatype-work
    chown -R nexus:root nexus3/
    exit
    ```

7. Prepare certificate using keytool (self-signed)

    ```
    su - nexus
    cd /opt/sonatype-work/nexus3
    keytool -keystore etc/ssl/keystore.jks -storepass password -genkeypair -keyalg RSA -validity 3653 -dname "CN=nexus" -keysize 4096 -storetype pkcs12 -alias jetty -ext "SAN=IP:10.0.19.246"
    keytool -exportcert -alias jetty -keystore etc/ssl/keystore.jks -storepass password -rfc -file jetty.pem
    exit
    ```

    The following are the changeable parameters:
    - **Required**: IP address of the node in the SAN extension 
    - Certificate validity (set for 10 years)
    - Distinguished name (set to `CN=nexus`)
    - Keystore password (set to `password`)
    - Certificate alias name (set to `jetty`)
    - Output file to store the certificate (set to `jetty.pem`)

8. Add certificate to ubuntu ca trust

    ```
    sudo mkdir /usr/local/share/ca-certificates/nexus
    sudo cp /opt/sonatype-work/nexus3/jetty.pem /usr/local/share/ca-certificates/nexus/jetty.crt
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

7. Customize nexus configuration (as root):

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
    **Notes**:

    - The `jetty-http.xml` can eventually be removed once HTTPS is verified
    - The certAlias and password parameters can be changed depending on how you build the keystore.jks in step 7.

8. Start nexus

    ```
    sudo systemctl start nexus
    ```

9. Get the initial nexus admin password

    ```
    cat /opt/sonatype-work/nexus3/admin.password
    ```
    Login to the nexus at `http://<ipaddress>:8081/` using the user `admin` and its password. The password will have to be changed on the initial login.

10. In the nexus Web interface, select the GearBox icon (**Settings**) > **Repositories**. Click **Create Repository** > `docker (hosted)`, specify the Name, check the HTTPS and use port 8444 and click **Crete Repository**.  

10. Setup openshift cli:

    ```
    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.6.28/openshift-client-linux.tar.gz
    tar -xf openshift-client-linux.tar.gz
    sudo mv oc /usr/local/bin/oc
    sudo mv kubectl /usr/local/bin/kubectl
    ```

11. Setup pull-secret - create pull-secret.txt from the content in [https://cloud.redhat.com/openshift/install/pull-secret](https://cloud.redhat.com/openshift/install/pull-secret)

    - Base64 encode your repository access:

        ```
        echo -n 'admin:<newpassword>' | base64
        ```

    - Append the access to your repo in the pull secret (use the actual IP adn authentication values): 

        ```
        ,"<ip>:8444":{"auth":"<base64encoded auth>","email":"a@example.com"}
        ```
    - Verify the generated file using `jq` (fix any error the comes):

        ```
        cat pull-secret.txt | jq .
        ```

12. Run mirror:

    ```
    export OCP_RELEASE=4.6.28 
    export LOCAL_REGISTRY='<ipaddr>:8444' 
    export LOCAL_REPOSITORY='ocp46' 
    export PRODUCT_REPO='openshift-release-dev' 
    export LOCAL_SECRET_JSON='pull-secret.txt' 
    export RELEASE_NAME="ocp-release" 
    export ARCHITECTURE="x86_64"
    ```
    **Note**: change the appropriate OCP_RELEASE, LOCAL_REGISTRY and LOCAL_REPOSITORY values

    ```
    oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}
    ```
