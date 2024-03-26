# Enterprise-Authentication by Kerberos
## Content Page

1. [Objective](#objective)
2. [Environment Setup Information](#environment-setup-information)
3. [Setting Up Docker Containers](#setting-up-docker-containers)
4. [Configuring Kerberos server in the server container](#configuring-kerberos-server-in-the-server-container)
5. [Configuring Kerberos database](#configuring-kerberos-database)
   - Create the Kerberos Database
   - Managing Principals with kadmin
   - Adding User Principals
   - Adding Host Principals
   - Generating Keytab Files
   - Copying Keytab Files
   - Restarting Services
6. [Setting up client within client container](#setting-up-client-within-client-container)
   - Install Kerberos Client Packages
   - Verify Kerberos Client Configuration
7. [Setting up SSH-Server within ssh-server container](#setting-up-ssh-server-within-ssh-server-container)
   - Update Kerberos Configuration in `krb5.conf`
   - Create User Account
   - Configure OpenSSH for Kerberos Authentication
   - Restart SSH Service
   - Copy and Set Permissions for `krb5.keytab` File
8. [Testing SSH Kerberos authentication as client](#testing-ssh-kerberos-authentication-as-client)
   - Request a Ticket-Granting Ticket (TGT)
   - Verify TGT Granting
   - Authenticate with SSH Server
   - Verify TGT and Service Tickets
9. [Setting up Apache Server Within Apache server container](#setting-up-apache-server-within-apache-server-container)
   - Update Kerberos Configuration in `krb5.conf`
   - Install the `mod_auth_kerb` Authentication Module
   - Enable the Module in Apache
   - Copy and Set Permissions for Keytab File
   - Configure Apache for Kerberos Authentication
   - Create a Test File for Kerberos Authentication
   - Restrict Access for Kerberos Principals
   - Restart Apache Service
10. [Testing Apache Kerberos authentication as client](#testing-apache-kerberos-authentication-as-client)
      - Request a Ticket-Granting Ticket (TGT)
      - Verify TGT Granting
      - Access Restricted Web Directory
      - Verify TGT and Service Tickets

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
### Define containers:
create a directory as follows:
    
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
    
Configure the files as follows. Remember to replace XXXX with the PIN code decided before.

Docker-compose.yaml

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

Client Dockerfile
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

Server Dockerfile
    
    
    #server/Dockerfile
    
    FROM ubuntu:18.04
    RUN mkdir -p /run/systemd && echo 'docker' > /run/systemd/container
    RUN apt-get update -y --fix-missing
    RUN apt-get install nano rsyslog -y
    RUN apt install inetutils-ping -y
    RUN apt-get install openssh-server -y

SSH-server Dockerfile
    
    
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

Apache-server Dockerfile
    
    
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

After the configuration, we can execute to compose and build docker containers.
<p align="center">
  <img width="274" alt="docker-compose" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/587022bb-48d9-4ed3-982e-5e2df4eee8fd">
</p>
    
This will spawn four containers - kerberos_client2, kerberos_server, ssh-server, and apache-server - with pre-assigned static IP address and DNS resolution.
Execute $docker ps to check the situation of the four containers:
<p align="center">
  <img width="472" alt="docker-ps" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/697610e2-3bfb-4964-90b1-320bff828aa6">

</p>


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
<p align="center">
  <img width="469" alt="krb5kdc-config" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/498bf5fe-5296-4f17-bdcd-76dd5604ebc6" align="center">
  
</p>
If you need to start over or make changes to the configurations later on, you have a couple of options. You can either edit the configuration files directly (we'll cover this in the next part), or you can use the command:

```bash
$ sudo dpkg-reconfigure --no-reload krb5-config
```

Your **`/etc/krb5.conf` will look like this:**
<p align="center">
  <img width="437" alt="krb5-config" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/65e0ba6e-b30f-448b-8086-2615feeb39dc" align="center"  >
  
</p>
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
    
   <p align="center">
     <img width="625" alt="addprinc" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/013a1db5-5fc4-4a5d-80ae-bf881935bbc4">

   </p>
    
    Adding user principals allows individuals to authenticate themselves to the Kerberos system.
    
4. **Adding Host Principals:**
    
    Add host principals for both your SSH server and Apache server using the commands below:
    
    ```
    $ addprinc -randkey host/ssh-server.kerberosprojectXXXX.com@kerberosprojectXXXX.com
    $ addprinc -randkey HTTP/apache.kerberosprojectXXXX.com@kerberosprojectXXXX.com
    ```
    <p align="center">
      <img width="667" alt="2" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/41c81fc3-f1b6-43c5-9c9a-c6bf5819e406">

    </p>
    
    Services like SSH and Apache are enabled to authenticate themselves to the Kerberos system.
    
    Execute `$listprincs` to check if all the required principals are added successfully:
    
   <p align="center">
     <img width="441" alt="3" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/d81a1d5e-8da9-4663-92b0-4e769bfe9654">

   </p>
    
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
    
    Restart the `krb5-kdc` and `krb5-admin-server` services and verify their status by the following commands:
    ```
    $ sudo systemctl restart krb5-kdc
    $ sudo systemctl restart krb5-admin-server
    ```
    
     They should be active and running to ensures that any changes made to the Kerberos configuration take effect.

## Setting up client within client container
Get into the client container and configure the Kerberos client within your Docker container by following these steps:

1. **Install Kerberos Client Packages:**
    
    Install the necessary Kerberos client packages on the Docker container by the following command:
    
    ```
    $ apt-get install krb5-user krb5-config
    ```
    
    Installing these packages allows the Docker container to interact with the Kerberos server for authentication purposes.
    
2. **Verify Kerberos Client Configuration:**
    
    After installation, it's important to ensure that the client's Kerberos configuration is correct. 
    
    Review the `/etc/krb5.conf` file within the container and confirm that the default realm and KDC/admin server information match the settings required for this project.
    
    Your `/etc/krb5.conf` file should look like this:
    <p align="center">
      <img width="436" alt="4" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/9f26f425-625a-45b6-8732-3b3a0c9262e6">

    </p>
    
    This is to ensure that the client can communicate effectively with the Kerberos server for authentication and ticket-granting purposes.
## Setting up SSH-Server within ssh-server container
Get into the ssh-server container to set up and kick off the service.

1. **Update Kerberos Configuration in `krb5.conf`:**
    
    Open the `/etc/krb5.conf` file in the SSH-server container. Change the configuration files with the following details:
    
    ```
    [libdefaults]
    default_realm = kerberosprojectXXXX.com
    
    [realms]
    kerberosprojectXXXX.com = {
    kdc_ports = 88,750
    kadmind_port = 749
    kdc = server.kerberosprojectXXXX.com
    admin_server = server.kerberosprojectXXXX.com
    }
    
    ```
    
    This is to ensure that the SSH server container is properly configured to communicate with the Kerberos server for later authentication.
    
2. **Create User Account:**
    
    Create a user account on the SSH-server container using the first name and set up a password for this account. Ensure you can log into the server container as this user.
    
3. **Configure OpenSSH for Kerberos Authentication:**
    
    Edit the `/etc/ssh/sshd_config` file to enable Kerberos authentication. 
    
    Update the Kerberos and GSSAPI flags. Set `KerberosTicketCleanup` to `no` to retain the token value as proof within the Kerberos authentication log.
    
    Your `/etc/ssh/sshd_config` file should look like this:
    
    KerberosAuthentication yes
    GSSAPIAuthentication yes
    KerberosTicketCleanup no
    
4. **Restart SSH Service:**
    
    Once satisfied with the changes, restart the SSH service to apply the configuration updates.
    
5. **Copy and Set Permissions for `krb5.keytab` File:**
    
    Copy the `ssh-server.keytab` file from your local host machine to the SSH-server container, placing it in the `/etc` folder. 
    
    Remember to exit and return to your VM before executing $ sudo docker cp ./ssh-server.keytab ssh-server:/etc
    
    Rename the file as `krb5.keytab` and chmod 600 for it.
    
    The permission of it will be like this:
    <p align="center">
      <img width="321" alt="5" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/cc63ab72-a735-46ea-a449-982cb0bbfce6">

    </p>
    
    This keytab file contains encrypted keys used for Kerberos authentication between the SSH server and the Kerberos system, enhancing security and authentication reliability.
    

By following these steps, proper communication between the SSH server and the Kerberos server will be set up for Kerberos authentication.
## Testing SSH Kerberos authentication as client
It’s time to test the SSH Kerberos authentication. Let’s get into the client-server container.

1. **Request a Ticket-Granting Ticket (TGT):**
    
    Request a TGT from the Key Distribution Center (KDC) for the user principal `<first_name>` created beforehand:
    
    ```
    $ kinit <first_name>
    ```
    
2. **Verify TGT Granting:**
    
    After requesting the TGT, verify that it has been granted by checking the ticket cache using:
    
    ```
    $ klist
    ```
    
    The ticket cache should look like this:
    
   <p align="center">
     <img width="530" alt="6" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/eb632205-dce7-4be2-91f3-ed4f44c28044">

   </p>
    
    This is to ensure that the authentication process was successful and the TGT is available for use in subsequent authentication processes.
    
3. **Authenticate with SSH Server:**
    
    From the client container, authenticate with the SSH server hosted in the server container using the command:
    
    ```
    $ ssh -vvv <first_name>@ssh-server.kerberosprojectXXXX.com
    ```
    
    This command establishes a connection to the SSH server using Kerberos/GSSAPI authentication. No password is required since the Kerberos service ticket is used for authentication.
    
    Your terminal should look like this indicating a successful SSH connection.
    
   <p align="center">
     <img width="429" alt="7" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/9f28e1d3-79cd-4816-bb87-c3277ab948b8">

   </p>
    
4. **Verify TGT and Service Tickets:**
    
    After authentication, verify if the TGT and Service tickets have been granted and stored in the client container by running:
    
    ```
    $ klist
    ```
    you will get things like this:
   <p align="center">
     <img width="586" alt="8" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/05e9abf9-c89a-4d9a-922f-99f1dabdd4e9">

   </p>
## Setting up Apache Server Within Apache server container
Get into the apache server container to set up web service for Kerberos authentication.

1. **Update Kerberos Configuration in `krb5.conf`:**
    
    Open the `/etc/krb5.conf` file in the Apache server container and replace its contents with the following configuration:
    
    ```
    [libdefaults]
    default_realm = kerberosprojectXXXX.com
    
    [realms]
    kerberosprojectXXXX.com = {
    kdc_ports = 88,750
    kadmind_port = 749
    kdc = server.kerberosprojectXXXX.com
    admin_server = server.kerberosprojectXXXX.com
    }
    ```
    
2. **Install the `mod_auth_kerb` Authentication Module:**
    
    Install the `libapache2-mod-auth-kerb` package to enable the Kerberos authentication module for Apache:
    
    ```
    $ apt-get install libapache2-mod-auth-kerb
    ```
    
3. **Enable the Module in Apache:**
    
    The `mod_auth_kerb` module should be enabled automatically when the package is installed. If not, you can manually enable it by adding the following directive to the `apache2.conf` file:
    
    ```
    LoadModule auth_kerb_module /usr/lib/apache2/modules/mod_auth_kerb.so
    ```
    
4. **Copy and Set Permissions for Keytab File:**
    
    Copy the generated keytab file to your Apache server and provide proper permissions:
    
    ```
    $ chown root.www-data apache.keytab
    $ chmod 640 /etc/apache.keytab
    ```
    
    `www-data` typically used for web server processes. In many Linux systems, the Apache web server runs its processes using the `www-data` user and group.
    
    The permission for Keytab file will be like this:
    <p align="center">
      <img width="345" alt="9" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/84ce3404-c677-49d5-a2d0-1cced32ef9a7">

    </p>
    
5. **Configure Apache for Kerberos Authentication:**
    
    Edit the `/etc/apache2/sites-enabled/000-default.conf` file to include the following configuration:
    
    ```
    <Directory "/var/www/html/test">
    AuthType Kerberos
    AuthName "KERBEROS AUTHENTICATION"
    KrbAuthRealms kerberosprojectXXXX.com
    Krb5Keytab /etc/apache.keytab
    KrbMethodNegotiate On
    KrbMethodK5Passwd Off
    Require user <second user>@kerberosprojectXXXX.com
    </Directory>
    ```
    
    Remember to restrict the user allowed to access the website through kerberos authentication.
    
    This configuration specifies that Apache should use Kerberos authentication for the `/var/www/html/test` directory.
    
6. **Create a Test File for Kerberos Authentication:**
    
    Create a file to check if Kerberos authentication is working:
    
    ```
    $ mkdir /var/www/html/test
    $ echo "Restricted Access to Kerberos Principals \\n <XXXX>" >/var/www/html/test/restricted.test
    ```
    
    This test file allows you to verify that Kerberos authentication is functioning correctly for the specified directory.
    
    Don’t forget to edit the permission of the test file. The permission of it should look like this:
    <p align="center">
      <img width="419" alt="10" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/8f318443-4676-43f3-9029-abf449a8f49b">

    </p>
    
7. **Restrict Access for Kerberos Principals:**
    
    Restrict access to all Kerberos principals and allow only specific users to access the website test directory by updating the `Require` directive.
    
8. **Restart Apache Service:**
    
    Restart the Apache service to apply the changes:
   ```
   $ systemctl restart apache2
   ```
## Testing Apache Kerberos authentication as client
Again, get into the client server container and start testing the apache kerberos authentication.

1. **Request a Ticket-Granting Ticket (TGT):**
    
    Request a TGT from the Key Distribution Center (KDC) for the user principal `<first_name>` created before.
    
    ```
    $ kinit <first_name>
    ```
    
2. **Verify TGT Granting:**
    
    After requesting the TGT, verify that it has been granted by checking the ticket cache using:
    
    ```
    $ klist
    ```
    
3. **Access Restricted Web Directory:**
    
    Attempt to access the restricted web directory `/test` on the Apache server from the client container using the command:
    
    ```
    $ curl "<http://apache.kerberosprojectXXXX.com/test/restricted.test>" -u : --negotiate
    ```
    
    This command tests whether the user is authorized to access the restricted directory using Kerberos authentication. 
    
    Since user principal `<first_name>` is restricted in the previous setting. Thus, an error message was returned indicating unauthorized access.
    
    <p align="center">
      <img width="545" alt="11" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/9abbebe6-306c-46d2-a991-334896fbcc08">

    </p>
    
    Now, request TGT for user principal `<second_name>`  and try to access the restricted test html. It should be accessed successfully and return things like this:
    <p align="center">
      <img width="546" alt="12" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/e1616551-8be2-4b06-8864-b2024f1b69a5">

    </p>
    
4. **Verify TGT and Service Tickets:**
    
    After authentication attempts, verify if the TGT and Service tickets have been granted and stored in the client container by running:
    
    ```
    $ klist
    ```
    <p align="center">
      <img width="570" alt="13" src="https://github.com/jenniferwingna/Enterprise-Authentication-by-Kerberos/assets/116328799/451358fb-afc5-4e8f-ba10-bf1dc2b916d1">

    </p>
    
    From the screenshot, we can find that:
    
    A Ticket Granting Ticket (TGT) for **`kerberosproject6424.com`** was issued, allowing the user to request service tickets for accessing various services within the Kerberos realm.
    
    The subsequent lines show Service Principal Tickets issued for the HTTP service (`HTTP/apache.kerberosproject6424.com@kerberosproject6424.com`). These tickets allow the user to access specific services provided by the HTTP server within the specified realm.
    
    The "renew until" timestamp indicates the time until which the tickets can be automatically renewed for continued access without requiring the user to re-authenticate.

