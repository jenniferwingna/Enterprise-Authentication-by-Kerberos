# Enterprise-Authentication by Kerberos

## Objective
This project will dive into Kerberos architecture and explore how it operates in a real-world context. In this blog post, I will explain how Kerberos works through hands-on implementation.
Typically, a Kerberos setup involves a minimum of three distinct machines, which are Key Distribution Center(KDC), Ticket Granting Service(TGS), and Client Machine.
<p align="center">
  <img src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/d9c5d4a4-63da-4e51-8f0a-7c861f7d597a.jpg" width="700">
</p>
For the sake of simplicity, only two services are being used as examples - the OpenSSH Server and Apache Server. We can focus on the demonstration of Kerberos authentication and access control mechanisms. In a real-world scenario, a Kerberos setup would normally involve multiple servers and machines across different domains or organizational units.

One more thing, we will be using Docker to package and deploy applications and services without getting bogged down by infrastructure complexities.

## Environment Setup Information
A Kerberos realm will be set up with a four-digit PIN code(XXXX). The PIN code can be any combination of four numbers. Don’t forget to replace the XXXX with your PIN code:
1. Kerberos Realm: **kerberosprojectXXXX.com**
2. Kerberos Server for Realm/Administrative server: **server.kerberosprojectXXXX.com**
3. Administrative Server for Realm: **server.kerberosprojectXXXX.com**
4. Client domain name: **client.kerberosprojectXXXX.com**
5. User principal: **<your_first_name>**
6. Host/service principal: **host/server.kerberosprojectXXXX.com**
7. ssh-server: **host/ssh-server.kerberosprojectXXXX.com**
8. apache server: **host/apache.kerberosprojectXXXX.com**

## Setting Up Docker Containers
#### Define containers:
    
    create a directory as follows:
    
    ```markdown
    
    ├── kerberos
    │   ├── client
    │   │   └── Dockerfile
    │   ├── docker-compose.yaml
    │   └── server
    │       └── Dockerfile
    │   └── ssh-server
    │       └── Dockerfile
    │   └── apache-server
    │       └── Dockerfile
    ```
    
    Configure the files as follows. Remember to replace XXXX with the PIN code decided before.
     
    Docker-compse.yaml
    
    ```markdown
    #docker-compose.yaml
    version: '3.3'
    services:
      server:
        container_name: kerberos_server
        hostname: server.kerberosprojectXXXX.com
        build:
          context: ./server
        tty: true
        networks:
          vpcbr2:
            ipv4_address: 10.6.0.5
        extra_hosts:
          - "server.kerberosprojectXXXX.com:10.6.0.5"
          - "client.kerberosprojectXXXX.com:10.6.0.6"
          - "ssh-server.kerberosprojectXXXX.com:10.6.0.4"
          - "apache.kerberosprojectXXXX.com:10.6.0.3"
      client:
        container_name: kerberos_client2
        hostname: client.kerberosprojectXXXX.com
        build:
          context: ./client
        tty: true
        networks:
          vpcbr2:
            ipv4_address: 10.6.0.6
        extra_hosts:
          - "server.kerberosprojectXXXX.com:10.6.0.5"
          - "client.kerberosprojectXXXX.com:10.6.0.6"
          - "ssh-server.kerberosprojectXXXX.com:10.6.0.4"
          - "apache.kerberosprojectXXXX.com:10.6.0.3"
      ssh-server:
        container_name: ssh-server
        hostname: ssh-server.kerberosprojectXXXX.com
        build:
          context: ./ssh-server
        networks:
          vpcbr2:
           ipv4_address: 10.6.0.4
        extra_hosts:
          - "server.kerberosprojectXXXX.com:10.6.0.5"
          - "client.kerberosprojectXXXX.com:10.6.0.6"
          - "ssh-server.kerberosprojectXXXX.com:10.6.0.4"
          - "apache.kerberosprojectXXXX.com:10.6.0.3"
        tty: true
      apache-server:
        container_name: apache-server
        hostname: apache.kerberosprojectXXXX.com
        build:
          context: ./apache-server
        networks:
          vpcbr2:
            ipv4_address: 10.6.0.3
        extra_hosts:
        - "server.kerberosprojectXXXX.com:10.6.0.5"
        - "client.kerberosprojectXXXX.com:10.6.0.6"
        - "ssh-server.kerberosprojectXXXX.com:10.6.0.4"
        - "apache.kerberosprojectXXXX.com:10.6.0.3"
        tty: true
    networks:
      vpcbr2:
        driver: bridge
        ipam:
          config:
            - subnet: 10.6.0.0/16
    ```
    
    Client Dockerfile
    
    ```markdown
    #client/Dockerfile
    
    FROM ubuntu:18.04
    RUN mkdir -p /run/systemd && echo 'docker' > /run/systemd/container
    RUN #(nop) CMD ["/bin/bash"]
    RUN apt-get update --fix-missing && apt-get install -y openssh-server
    RUN apt-get install -y iputils-ping
    RUN apt-get install -y sudo nano
    RUN mkdir /var/run/sshd
    RUN echo "root:toor" | chpasswd
    RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
    RUN echo "export VISIBLE=now" >> /etc/profile
    RUN #(nop) EXPOSE 22
    RUN #(nop) CMD ["/usr/sbin/sshd" "-D"]
    ```
    
    Server Dockerfile
    
    ```markdown
    #server/Dockerfile
    
    FROM ubuntu:18.04
    RUN mkdir -p /run/systemd && echo 'docker' > /run/systemd/container
    RUN apt-get update -y --fix-missing
    RUN apt-get install nano rsyslog -y
    RUN apt install inetutils-ping -y
    RUN apt-get install openssh-server -y
    ```
    
    SSH-server Dockerfile
    
    ```markdown
    #ssh-server/Dockerfile
    
    FROM ubuntu:18.04
    EXPOSE 749 88 22
    RUN mkdir -p /run/systemd && echo 'docker' > /run/systemd/container
    RUN apt-get update -y --fix-missing
    RUN apt-get -qq install locales krb5-kdc krb5-admin-server
    RUN apt-get -qq install openssh-server
    RUN apt-get -qq install vim 
    RUN apt-get install nano rsyslog -y
    RUN apt install inetutils-ping -y
    RUN apt-get -qq clean
    ```
    
    Apache-server Dockerfile
    
    ```markdown
    #apache-server/Dockerfile
    
    FROM ubuntu:18.04
    EXPOSE 749 88 80
    RUN mkdir -p /run/systemd && echo 'docker' > /run/systemd/container
    RUN apt-get update -y --fix-missing
    RUN apt-get -qq install locales krb5-kdc krb5-admin-server
    RUN apt-get -qq install apache2
    RUN apt-get -qq install vim 
    RUN apt-get install nano rsyslog -y
    RUN apt install inetutils-ping -y
    RUN apt-get -qq clean
    ```
    
    After the configuration, we can execute to compose and build docker containers.
    
    <img width="274" alt="docker-compose" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/587022bb-48d9-4ed3-982e-5e2df4eee8fd">
    
      This will spawn four containers - kerberos_client2, kerberos_server, ssh-server, and apache-server - with pre-assigned static IP address and DNS resolution.
      
      Execute $docker ps to check the situation of the four containers:
    <img width="472" alt="docker-ps" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/697610e2-3bfb-4964-90b1-320bff828aa6">

## Configuring Kerberos server in the server container
To set up the Key Distribution Center (KDC) and the admin server, we'll  get into the server container by this command:

```bash
$ sudo docker exec -it kerberos_server bash
```

Then, install the associated packages on the server container by this command:

```bash
$ apt install krb5-kdc krb5-admin-server
```

During the installation, you'll be prompted to provide some configuration details. Don't worry if the installation shows a failed status; this is expected at this stage.

Responding to the installation questions will help us configure the KDC and admin server properly.

After the installation, it's important to verify the default configurations generated based on your responses. 

You can review the configuration file located at **`/etc/krb5kdc/kdc.conf`**. Make sure that the realm name specified in the file matches the realm name you intend to use for this project.

Your kdc.conf will look like this:

<img width="469" alt="krb5kdc-config" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/498bf5fe-5296-4f17-bdcd-76dd5604ebc6" align="center">


If you need to start over or make changes to the configurations later on, you have a couple of options. You can either edit the configuration files directly (we'll cover this in the next part), or you can use the command:

```bash
$ sudo dpkg-reconfigure --no-reload krb5-config
```

Your **`/etc/krb5.conf` will look like this:**

<img width="437" alt="krb5-config" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/65e0ba6e-b30f-448b-8086-2615feeb39dc" align="center"  >


Remember to restart the services after making any changes, especially when editing the configuration files manually.

## Configuring Kerberos database
Stay in the server container, we will be configuring the kerberos database. In Kerberos, a database is used to store information about principals, which represent users and services on the network.

1. **Create the Kerberos Database:**
    
    Use the command below to create the database for your realm:
    
    ```
    $ krb5_newrealm
    ```
    
    The Kerberos database is initialized for your realm, allowing you to store user and service principals securely.
    
2. **Managing Principals with kadmin:**
    
    The `kadmin` tool is used to manage the Kerberos database. Access the kadmin console using the following command:
    
    ```
    $ kadmin.local
    ```
    
3. **Adding User Principals:**
    
    Inside the kadmin console, add two user principals, replacing `<first_user>` with the desired names:
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/9e0c49bd-874e-4a4c-9f49-63ff87d1aaf2/07a16b97-bef8-41a8-bc0c-fc77fc682705/Untitled.png)
    
    Adding user principals allows individuals to authenticate themselves to the Kerberos system.
    
4. **Adding Host Principals:**
    
    Add host principals for both your SSH server and Apache server using the commands below:
    
    ```
    $ addprinc -randkey host/ssh-server.kerberosprojectXXXX.com@kerberosprojectXXXX.com
    $ addprinc -randkey HTTP/apache.kerberosprojectXXXX.com@kerberosprojectXXXX.com
    ```
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/9e0c49bd-874e-4a4c-9f49-63ff87d1aaf2/a06f7999-f37a-4eb3-9bf1-43336ac8152c/Untitled.png)
    
    Services like SSH and Apache are enabled to authenticate themselves to the Kerberos system.
    
    Execute $listprincs to check if all the required principals are added successfully:
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/9e0c49bd-874e-4a4c-9f49-63ff87d1aaf2/24322431-7158-4c18-bd5d-340688cde22f/Untitled.png)
    
5. **Generating Keytab Files:**
    
    Generate keytab files for the SSH server and Apache server using the commands below:
    
    ```
    $ ktadd -k /etc/ssh-server.keytab host/ssh-server.kerberosprojectXXXX.com@kerberosprojectXXXX.com
    $ ktadd -k /etc/apache.keytab HTTP/apache.kerberosprojectXXXX.com@kerberosprojectXXXX.com
    ```
    
    Keytab files contain encrypted keys used for authentication between services and the Kerberos system.
    
6. **Copying Keytab Files:**
    
    Copy the generated keytab files (`ssh-server.keytab` and `apache.keytab`) from the server to your host machine for further use.
    
7. **Restarting Services:**
    
    Restart the `krb5-kdc` and `krb5-admin-server` services and verify their status by following command:
    
    $ sudo systemctl restart krb5-kdc
    $ sudo systemctl restart krb5-admin-server
    
     They should be active and running to ensures that any changes made to the Kerberos configuration take effect.
      
