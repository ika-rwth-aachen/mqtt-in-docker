# mqtt-in-docker

This repository contains instructions and configuration files that help you to set up and use an [MQTT broker](https://en.wikipedia.org/wiki/MQTT#MQTT_broker) and MQTT Clients in [Docker](https://docker.com/). We will explain how **Docker-based clients** can communicate with a **Docker-based broker** using **authentication** and **encryption**. Instructions are provided on how to **set up and test** the software **locally** and **across multiple machines**. We describe a setup using the broker [Mosquitto](https://mosquitto.org/). The setup can in principle be adjusted to [other MQTT brokers](https://mqtt.org/software/) as well.

## Preface

The authors of this repository are in no way affiliated with the [Eclipse Foundation](https://www.eclipse.org/), the maintainers of [Mosquitto](https://mosquitto.org/), [OASIS Open](https://oasis-open.org), the maintainers of a standard for the [MQTT Protocol](https://mqtt.org), or [Docker Inc.](https://www.docker.com/), the company that develops Docker. The goal of this repository is to help developers get started with MQTT in Docker.

**Warning**: The setups described in this repository can be *insecure* if handled incorrectly. Usage happens at your own risk.

## Content
- [Repository Structure](#repository-structure)
- [Motivation and Background](#motivation-and-background)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Using your own Configuration](#using-your-own-configuration)
- [Authenticated Communication](#authenticated-communication)
  - [Creating a Password File](#creating-a-password-file)
  - [Defining a Broker Configuration for Authenticated Communication](#defining-a-broker-configuration-for-authenticated-communication)
  - [Testing Authenticated Communication](#testing-authenticated-communication)
- [Encrypted Communication](#encrypted-communication)
  - [Defining a Broker Configuration for Encrypted Communication](#defining-a-broker-configuration-for-encrypted-communication)
  - [Implementing a Public-Key Infrastructure using OpenSSL](#implementing-a-public-key-infrastructure-using-openssl)
    - [Creating a Certificate Authority](#creating-a-certificate-authority)
    - [Creating a Signed Server Certificate](#creating-a-signed-server-certificate)
    - [Creating a Signed Client Certificate](#creating-a-signed-client-certificate)
  - [Testing Encrypted Communication](#testing-encrypted-communication)
- [Creating your own MQTT and Docker-based Applications](#creating-your-own-mqtt-and-docker-based-applications)
- [Feedback](#feedback)

## Repository Structure

```
mqtt-docker
├── authentication              # where password files can be stored  
├── configs                     # where Mosquitto configuration files can be stored
└── encryption                  # files for the Public-Key Infrastructure
    ├── certificate-authority       # certificates, keys and certificate signing requests for the certificate-authority
    ├── client                      # certificates, keys and certificate signing requests for clients
    └── server                      # certificates, keys and certificate signing requests for the server
```

## Motivation and Background

[**MQTT**](https://mqtt.org/) is a popular protocol for the communication between multiple machines in the [**Internet of Things (IoT)**](https://en.wikipedia.org/wiki/Internet_of_things). It is used in many industries such as Automotive, Logistics, Manufacturing, Robotics, and Smart Home. The protocol uses the publish / subscribe pattern to connect devices to each other. Connections are handled by an MQTT broker, which distributes messages published on topics by senders to receivers that subscribe to these topics.

[**Docker**](https://docker.com/) comprises a set of tools for the virtualization and modularization of software using so-called containers. **Containers** are fully functional computing environments for individual applications. They make applications easily portable and allow them to be run in isolation in any cloud or non-cloud environment. Containers can be orchestrated using tools such as [Kubernetes](https://kubernetes.io).

**Advantages** of setting up MQTT-based applications in Docker comprise the benefits of containerization in general: 
- rapid application deployment,
- easy (over-the-air) updates,
- reduced compatibility issues,
- portability across machines,
- component reuse,
- application sharing with others,
- lightweight footprint,
- minimal overhead,
- simplified maintenance.

Providing Docker containers with MQTT communication capabilities adds a lightweight communication protocol to Docker-based applications, further fostering their use in the Internet of Things.

Especially if **authentication and encryption** are to be ensured, it can be cumbersome to set up Docker-based applications communicating via MQTT. Both clients and brokers need to be part of a **Public-Key Infrastructure**, where digital certificates are used to facilitate a secure exchange of data. 

This repository therefore provides **step-by-step instructions** on how to 
- set up and configure different MQTT components in Docker,
- set up configurations for usernames and passwords to ensure authentication,
- set up a Public-Key Infrastructure to ensure encrypted communication,
- test all variations of the described setups and configurations.

For each step, some background information and links to additional resources are given, allowing the reader to gain a deeper understanding of the topic.

## Prerequisites

Make sure **Docker** is installed on your system. For installation instructions, refer to the [Docker docs](https://docs.docker.com/engine/install/ubuntu/) and the [post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/). If you want to use encryption, make sure that [OpenSSL](https://www.OpenSSL.org/) is installed on your system.

## Quick Start

The most essential component when setting up applications that communicate via MQTT is the MQTT broker. In the instructions of this repository, we will use [Mosquitto](https://mosquitto.org/) as an MQTT broker.

A simple way to start an Mosquitto MQTT broker in Docker is the following command:

```bash
docker run --rm --network host --name mosquitto eclipse-mosquitto
```

The above command uses the default Mosquitto config, which is already contained in the `eclipse-mosquitto` [Docker image](https://hub.docker.com/_/eclipse-mosquitto). A copy of that config can be found under [configs/mosquitto_default.conf](configs/mosquitto_default.conf). Read through this config to get information on all options for customization.

## Using your own Configuration

It is possible to use custom Mosquitto configs by using the `-v` flag in the `docker run` command to mount custom configuration files into the Docker container. The following command overwrites the default config, which is part of the `eclipse-mosquitto` Docker image, with the equivalent default config [mosquitto_default.conf](configs/mosquitto_default.conf) from this repository.

```bash
# mqtt-broker $
docker run \
    --rm \
    --network host \
    -v $PWD/configs/mosquitto_default.conf:/mosquitto/config/mosquitto.conf:ro \
    --name mosquitto \
    eclipse-mosquitto
```

You may create your own config, put it into [configs](configs) and replace `mosquitto_default.conf` in the above command with the name of your config.

## Authenticated Communication

A more secure way to use an MQTT broker is to require client authentication, i.e. username and password, before being allowed to connect to the broker.

We tell the broker which users and corresponding passwords are allowed to connect by providing an additional password file `mosquitto.passwd` found in [authentication](authentication). For now, the file is empty, but it can be filled as described below.

### Creating a Password File
We can use the tool [`mosquitto_passwd`](https://mosquitto.org/man/mosquitto_passwd-1.html) to manage password files for the Mosquitto broker. The tool is available in the `eclipse-mosquitto` Docker image. 

The password file can either be **appended** with a new user or **overwritten** with a new user, which removes all previously defined users.

<details>
<summary>To create a password file, click to expand and follow the instructions.</summary>

1. To **append** the `mosquitto.passwd` password file, replace `<username>` in the below command and run:

    ```bash
    # mqtt-broker $
    docker run \
        -it \
        --rm \
        -v $PWD/authentication/mosquitto.passwd:/mosquitto/config/mosquitto.passwd \
        --name mosquitto-add-user \
        --user "$(id -u):$(id -g)" \
        eclipse-mosquitto mosquitto_passwd /mosquitto/config/mosquitto.passwd <username>
    ```

2. To **overwrite** an existing `mosquitto.passwd` file (added flag `-c`), replace `<username>` in the below command and run:

    ```bash
    # mqtt-broker $
    docker run \
        -it \
        --rm \
        -v $PWD/authentication/mosquitto.passwd:/mosquitto/config/mosquitto.passwd \
        -v $PWD/authentication/mosquitto.passwd:/mosquitto/config/mosquitto.passwd.tmp \
        --name mosquitto-add-user \
        --user "$(id -u):$(id -g)" \
        eclipse-mosquitto mosquitto_passwd -c /mosquitto/config/mosquitto.passwd <username>
    ```

   - You will be asked to enter a password for the username twice. Afterwards, the Docker container used to change the password file automatically stops. The username and a [hash](https://en.wikipedia.org/wiki/Hash_function) of the password are stored in [configs/mosquitto.passwd](authentication/mosquitto.passwd).
   - **Hint**: In the above commands, we need to set the user id and the group id of the user in the container to your current user id and group id (`--user "$(id -u):$(id -g)"`), so you will continue to own the password file after changing it.
</details>

### Defining a Broker Configuration for Authenticated Communication

To start a broker which uses the password file filled above, we need a new config file which defines where to find the password file. This config file, which is provided as part of this repository at [authentication/mosquitto_authenticated.conf](configs/mosquitto_authenticated.conf), introduces a few additional changes:

- `password_file /mosquitto/config/mosquitto.passwd`: This tells the broker where to find the password file.
- `listener 1883`: The broker only listens on port 1883, the default port for unencrypted MQTT messages.
- (optional) `set_tcp_nodelay true`: It is usually beneficial to use this setting to disable Nagle's algorithm on client sockets of the broker. This has the effect of reducing the latency of individual messages at the potential cost of increasing the number of packets being sent (see [here](https://mosquitto.org/man/mosquitto-conf-5.html)).

### Testing Authenticated Communication

For a test of the authenticated communication between clients and the broker, in the following, we will create
1. an example password file ([mosquitto_passwd](https://mosquitto.org/man/mosquitto_passwd-1.html)), 
2. a broker secured by the password file ([mosquitto](https://mosquitto.org/man/mosquitto-8.html)), 
3. a client publishing data to the broker ([mosquitto_pub](https://mosquitto.org/man/mosquitto_pub-1.html)), and
4. a client subscribing to data from the broker ([mosquitto_sub](https://mosquitto.org/man/mosquitto_sub-1.html)).

All used tools ([mosquitto_passwd](https://mosquitto.org/man/mosquitto_passwd-1.html), [mosquitto](https://mosquitto.org/man/mosquitto-8.html), [mosquitto_pub](https://mosquitto.org/man/mosquitto_pub-1.html), and [mosquitto_sub](https://mosquitto.org/man/mosquitto_sub-1.html)) are already contained in the `eclipse-mosquitto` Docker image. In the Docker commands below, if no tool is named after the image name `eclipse-mosquitto`, `mosquitto` will be run by default and a broker is started.

<details>
<summary>To conduct this test, click to expand and follow the instructions.</summary>

1. First, overwrite the password file [configs/mosquitto.passwd](authentication/mosquitto.passwd) with the user `test_connection` and the password `test_connection`. Run the following command and enter `test_connection` when asked for the password (twice).
    ```bash
    # mqtt-broker $
    docker run \
        -it \
        --rm \
        -v $PWD/authentication/mosquitto.passwd:/mosquitto/config/mosquitto.passwd \
        --name mosquitto-add-user \
        --user "$(id -u):$(id -g)" \
        eclipse-mosquitto mosquitto_passwd -c /mosquitto/config/mosquitto.passwd test_connection
    ```


2. Run the MQTT broker and use the previously created password file by running this command:
    ```bash
    # mqtt-broker $
    docker run \
        --rm \
        --network host \
        --name mosquitto-authenticated \
        -v $PWD/configs/mosquitto_authenticated.conf:/mosquitto/config/mosquitto.conf:ro \
        -v $PWD/authentication/mosquitto.passwd:/mosquitto/config/mosquitto.passwd:ro \
        eclipse-mosquitto
    ```

3. In a new terminal, create a publisher which sends the message "`test_connection_message`" on the mqtt topic `"ping"` 20 times at 1 Hz. 
    ```bash
   docker run \
          --rm \
          --network host \
          --name test_mqtt_publish_authenticated \
          eclipse-mosquitto \
          mosquitto_pub \
            -u test_connection \
            -P test_connection \
            -h localhost \
            -t "ping" \
            --repeat 20 \
            --repeat-delay 1 \
            -m "test_connection_message"
    ```

    - The publisher authenticates with the broker using the username `-u test_connection` and the password `-P test_connection`.
    - **Multi-Machine Setup**: If you want to test the publisher in a setup across multiple machines, you may for example run the command above on another machine in the same network and replace `localhost` with the Host + Domain Name, or IP of the machine running the broker.
    - You may try to change either the username or the password in the above command. You should then see `Connection error: Connection Refused: not authorized.` This validates that the broker is indeed only accepting correctly authenticated users.

4. In a new terminal, create a subscriber which listens to messages on the topic `ping` and displays them.
    ```bash
    docker run \
          --rm \
          --network host \
          --name test_mqtt_subscribe_authenticated \
          eclipse-mosquitto \
          mosquitto_sub \
            -u test_connection \
            -P test_connection \
            -h localhost \
            -t "ping"
    ```
    - You should now see the `test_connection_message` in the terminal of the subscriber.
    - The subscriber again authenticates with the broker using a username `-u test_connection` and a password `-P test_connection`. Make sure the publisher has not finished sending messages yet when you run this command.
    - **Multi-Machine Setup**: If you want to test the subscriber in a setup across multiple machines, you may for example run the command above on another machine in the same network and replace `localhost` with the Host + Domain Name, or IP of the machine running the broker.
    
</details>

## Encrypted Communication

MQTT connections can be secured through [SSL/TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security). To establish the encrypted connection, TLS uses a [public key infrastructure (PKI)](https://en.wikipedia.org/wiki/Public_key_infrastructure).

### Defining a Broker Configuration for Encrypted Communication

To start a broker which allows encrypted communication, we need a new config file which defines where to find signed certificates and private keys. These config files, which are provided as part of this repository at [configs/\*.ssl\*conf](configs), introduce a few additional changes:

- `listener 8883`: The broker only listens on port 8883, the default port for encrypted MQTT messages.
- `require_certificate true`: This setting is used if we want to require clients to possess a signed certificate.
- `allow_anonymous true`: This setting is used if we want to only rely on the PKI and not use usernames and passwords.


## Implementing a Public-Key Infrastructure using OpenSSL

The example config files provided in this repository are already configured to read corresponding certificates and keys from the [encryption](encryption) directory. The certificates and keys do not exist yet though.

To create the necessary keys, certificates, and certificate signing requests, the [OpenSSL toolkit](https://www.openssl.org/) can be used. In particular, we will create
- a **Certificate Authority private key** (*ca-key.pem*), 
- a **Certificate Authority certificate signing request** (*ca-csr.pem*),
- use the latter to create a self-signed **Certificate Authority certificate** (*ca-cert.pem*), then create
- a **server private key** (*server-key.pem*) for the server running the broker,
- a **server certificate signing request** (*server-csr.pem*),
- use the latter to create a self-signed **server certificate** (*server-cert.pem*) and create
- a **CA serial number file** (*ca-cert.srl*) to keep track of signed certificates created by the CA, then create
- a **client private key** (*client-key.pem*) for a client trying to connect to the broker, 
- a **client certificate signing request** (*client-csr.pem*), and
- use the latter to create a self-signed **client certificate** (*client-cert.pem*)

The process will be explained in more detail below. Please follow the steps in the presented order.

### Creating a Certificate Authority

We start by creating a self-signed [root certificate](https://en.wikipedia.org/wiki/Root_certificate) of a root [Certificate Authority (CA)](https://en.wikipedia.org/wiki/Certificate_authority) and a corresponding [private key](https://en.wikipedia.org/wiki/Public-key_cryptography). The private key will follow the [X.509 standard](https://en.wikipedia.org/wiki/X.509) for [public key certificates](https://en.wikipedia.org/wiki/Public_key_certificate). The private key will also contain the public key of the CA, which can in principle be extracted from the former.

<details>
<summary>To create the CA key and CA certificate, click here to expand and follow the instructions.</summary>

1. In a first step, the **Certificate Authority private key** (*ca-key.pem*) and a **Certificate Authority certificate signing request** (*ca-csr.pem*) are created. All fields *may* be filled with the default value (press Enter), the [Common Name](https://www.ssl.com/faqs/common-name/) *must* be left empty. For the Organization Name, you may choose `ca`.  
    ```bash
    openssl req -newkey rsa:2048 -nodes -keyout encryption/certificate-authority/ca-key.pem -out encryption/certificate-authority/ca-csr.pem
    ```
    - `req`: We can create both in the same step using the *certificate request and certificate generating utility* [req](https://www.openssl.org/docs/man1.0.2/man1/openssl-req.html) provided by OpenSSL.
    - `rsa:2048`: The private key is generated using the [RSA algorithm](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) and has a size of 2048 bits. 
    - `-nodes`: The private key itself will not be encrypted. 
    - `*.pem`: Both the *ca-key* and the *ca-cert* will follow the [PEM format](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail). 

2. Next, the root **Certificate Authority certificate** (*ca-cert.pem*) will be created and signed using the previously generated CA certificate signing request (*ca-csr.pem*). 
    ```bash
    openssl x509 -req -in encryption/certificate-authority/ca-csr.pem -signkey encryption/certificate-authority/ca-key.pem -out encryption/certificate-authority/ca-cert.pem -days 365 -sha256
    ```
    - `x509`: We use the [certificate display and signing utility](https://www.openssl.org/docs/man1.0.2/man1/x509.html) provided by OpenSSL to create and sign the CA certificate.
    - `-days 365`: The CA certificate will be valid for 365 day . 
    - `-sha256`: The [SHA-256 hash function](https://en.wikipedia.org/wiki/SHA-2) will be used to generate the certificate.

</details>

### Creating a Signed Server Certificate

We will continue to create a **server private key** (*server-key.pem*) and a self-signed **server certificate** (*server-cert.pem*).

<details>
<summary>To create the server private key and the self-signed server certificate, click here to expand and follow the instructions.</summary>

1. Again, we create the **server certificate signing request** (*server-csr.pem*) in the same step as the **server private key** (*server-key.pem*). When asked for the Common Name, enter `localhost`. For the Organization Name, choose `server`. All other fields can be filled with the default value (press Enter). 

    ```bash
    openssl req -newkey rsa:2048 -nodes -keyout encryption/server/server-key.pem -out encryption/server/server-csr.pem
    ```
    - `req`: We will again be using the [certificate request and certificate generating utility](https://www.openssl.org/docs/man1.0.2/man1/openssl-req.html) provided by OpenSSL.
    - `rsa:2048`: For the **server private key** (*server-key.pem*), we again use the [RSA algorithm](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) to create the key with a size of 2048 bits.
    - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines, you should not use `localhost` for the Common Name but the Host + Domain Name, or IP of the machine running the broker.


2. The **server certificate signing request** (*server-key.pem*) is now handed to the CA (`-in encryption/server/server-csr.pem`), which creates a self-signed **server certificate** (`-out encryption/server/server-cert.pem`) using its **Certificate Authority certificate** (*ca-cert.pem*) and the corresponding **Certificate Authority private key** (*ca-key.pem*), with the help of the [certificate display and signing utility](https://www.openssl.org/docs/man1.0.2/man1/x509.html) provided by OpenSSL.
    ```bash
    openssl x509 -req -days 365 -in encryption/server/server-csr.pem -CA encryption/certificate-authority/ca-cert.pem -CAkey encryption/certificate-authority/ca-key.pem -CAcreateserial -out encryption/server/server-cert.pem
    ```
    - *Self-signed* means that we did not use a trusted public or commercial CA but the one we created ourselves earlier. 
    - We created a private key for the CA earlier that is separate from the CA's certificate. That is why we need to use `-CAkey encryption/certificate-authority/ca-key.pem` to describe the location of that private key.
    - We use `-CA encryption/certificate-authority/ca-cert.pem` to describe the location of the CA certificate to be used for signing.
    - Since this is the first time we sign a certificate other that the CA certificate, we set the flag `-CAcreateserial`. This will assign the serial number `01` to the signed certificate and a file `ca-cert.srl`, which contains the serial numbers of all previously created signed certificates. For additional certificates, we would need to use the option `-CAserial encryption/certificate-authority/ca-cert.srl` instead of `-CAcreateserial`, so the serial number of the additional certificates is incremented correctly. The *ca-cert.srl* would be updated and helps to keep track of all signed certificates created with the help of the CA.
</details>

### Creating a Signed Client Certificate

We can set up the broker such that it requires clients connecting to it to possess a **client certificate** (*client-cert.pem*) signed by the CA, and a **client private key** (*client-key.pem*) as well. The process for creating signed client certificates is the same as for the server. 

<details>
<summary>To create the client private key and the self-signed client certificate, click here to expand and follow the instructions.</summary>

1. First, we create a **client private key** and a **client certificate signing request**. In the setup described here, the broker will run on the same machine as the client, so you should choose `localhost` for the Common Name. For Organization Name, choose `client`. All other fields can be filled with the default value (press Enter).
    ```bash
    openssl req -newkey rsa:2048 -nodes -keyout encryption/client/client-key.pem -out encryption/client/client-csr.pem
    ```

    - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines, you should not use `localhost` for the Common Name, but the Host + Domain Name, or the IP of the machine running the broker.

2. The **client certificate signing request** (*client-key.pem*) is now handed to the CA (`-in encryption/client/client-csr.pem`), which creates a self-signed **client certificate** (`-out encryption/client/client-cert.pem`) using its **Certificate Authority certificate** (*ca-cert.pem*) and the corresponding **Certificate Authority private key** (*ca-key.pem*), with the help of the [certificate display and signing utility](https://www.openssl.org/docs/man1.0.2/man1/x509.html) provided by OpenSSL. 
    ```bash
    openssl x509 -req -days 365 -in encryption/client/client-csr.pem -CA encryption/certificate-authority/ca-cert.pem -CAkey encryption/certificate-authority/ca-key.pem -CAserial encryption/certificate-authority/ca-cert.srl -out encryption/client/client-cert.pem
    ```
    - Since this is not the first certificate signed by the CA, we need to use the option `-CAserial encryption/certificate-authority/ca-cert.srl` instead of `-CAcreateserial`, so the serial number of the additional certificate is incremented correctly. The *ca-cert.srl* is updated.
    - Now, we have created all necessary files to test the encrypted communication between clients and the broker.

</details>

### Testing Encrypted Communication

It is possible to 
-  only use the PKI to establish the connection between client and broker, or
-  use both the PKI and authentication through the password file ([authentication/mosquitto.passwd](authentication/mosquitto.passwd)).

In addition, it is possible to

- only require the server to be part of the PKI, or to
- require both server and client to be part of the PKI.

Depending on the combination you want to test, choose one of the below testing options a)-d), each using one of the config files found in the following table.

|    | **Only use the PKI to establish the connection between client and broker** | **Use both the PKI and authentication through the password file** |
|----|----|---|
| **Only require the server to be part of the PKI** |  a) `mosquitto.ssl-server.conf`  |  b) `mosquitto.ssl-all.conf`  |
| **Require both server and client to be part of the PKI** |  c) `mosquitto_authenticated.ssl-server.conf`  |  d) `mosquitto_authenticated.ssl-all.conf`  |

For a test of the encrypted communication between clients and the broker, for each of the options a)-d), we again create
1. an example password file ([mosquitto_passwd](https://mosquitto.org/man/mosquitto_passwd-1.html)) (only for communication authenticated by password file),
2. a broker (optionally) secured by the password file ([mosquitto](https://mosquitto.org/man/mosquitto-8.html)), 
3. a client publishing data to the broker ([mosquitto_pub](https://mosquitto.org/man/mosquitto_pub-1.html)), and
4. a client subscribing to data from the broker ([mosquitto_sub](https://mosquitto.org/man/mosquitto_sub-1.html)).

All used tools ([mosquitto_passwd](https://mosquitto.org/man/mosquitto_passwd-1.html), [mosquitto](https://mosquitto.org/man/mosquitto-8.html), [mosquitto_pub](https://mosquitto.org/man/mosquitto_pub-1.html), and [mosquitto_sub](https://mosquitto.org/man/mosquitto_sub-1.html)) are already contained in the `eclipse-mosquitto` Docker image. In the Docker commands below, if no tool is named after the image name `eclipse-mosquitto`, `mosquitto` will be run by default and a broker is started.

1. If you haven't already, and if you want to test encrypted communication authenticated (also) through a password file, first overwrite the password file [configs/mosquitto.passwd](authentication/mosquitto.passwd) with the user `test_connection` and the password `test_connection`. Run the following command and enter `test_connection` when asked for the password (twice).
    ```bash
    # mqtt-broker $
    docker run \
        -it \
        --rm \
        -v $PWD/authentication/mosquitto.passwd:/mosquitto/config/mosquitto.passwd \
        --name mosquitto-add-user \
        --user "$(id -u):$(id -g)" \
        eclipse-mosquitto mosquitto_passwd -c /mosquitto/config/mosquitto.passwd test_connection
    ```

2. Start a broker, a publisher and a subscriber to test the connection.
    
    <details>
    <summary>a) mosquitto.ssl-server.conf</summary>
  
    1. Start the broker.
        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --name mosquitto-encrypted \
            -v $PWD/configs/mosquitto.ssl-server.conf:/mosquitto/config/mosquitto.conf:ro \
            -v $PWD/encryption/server:/mosquitto/encryption/server:ro \
            -v $PWD/encryption/certificate-authority:/mosquitto/encryption/certificate-authority:ro \
            --user "$(id -u):$(id -g)" \
            eclipse-mosquitto
        ```
    2. Start the publisher in a new terminal.

        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --user "$(id -u):$(id -g)" \
            --name test_encrypted_publisher \
            -v $PWD/encryption/certificate-authority/ca-cert.pem:/mosquitto/encryption/certificate-authority/ca-cert.pem:ro \
            eclipse-mosquitto \
            mosquitto_pub \
                --cafile /mosquitto/encryption/certificate-authority/ca-cert.pem \
                -t "ping-encrypted" \
                -h localhost \
                --repeat 20 \
                --repeat-delay 1 \
                -p 8883 \
                -m "test_encrypted_connection_message"
        ```

        - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines, you should not use `localhost` in the command above, but the Host + Domain Name, or the IP of the machine running the broker. It must be the same as what you entered for the Common Name when you created the certificates. You should clone this repository on the other machine and must place the already created ca certificate (*ca-cert.pem*) into [encryption/certificate-authority](encryption/certificate-authority).
    
    3. Start the subscriber in a new terminal.

        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --user "$(id -u):$(id -g)" \
            --name test_encrypted_subscriber \
            -v $PWD/encryption/certificate-authority/ca-cert.pem:/mosquitto/encryption/certificate-authority/ca-cert.pem:ro \
            eclipse-mosquitto \
            mosquitto_sub \
                --cafile /mosquitto/encryption/certificate-authority/ca-cert.pem \
                -t "ping-encrypted" \
                -h localhost
        ```

        - You should now see the `test_encrypted_connection_message` in the terminal of the subscriber.
        - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines, you should not use `localhost` in the command above, but the Host + Domain Name, or the IP of the machine running the broker. It must be the same as what you entered for the Common Name when you created the certificates. You should clone this repository on the other machine and must place the already created ca certificate (*ca-cert.pem*) into [encryption/certificate-authority](encryption/certificate-authority).
    </details>

    <details>
    <summary>b) mosquitto.ssl-all.conf</summary>
    
    1. Start the broker.
        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --name mosquitto-encrypted \
            -v $PWD/configs/mosquitto.ssl-all.conf:/mosquitto/config/mosquitto.conf:ro \
            -v $PWD/encryption/server:/mosquitto/encryption/server:ro \
            -v $PWD/encryption/certificate-authority:/mosquitto/encryption/certificate-authority:ro \
            --user "$(id -u):$(id -g)" \
            eclipse-mosquitto
        ```
    2. Start the publisher in a new terminal.

        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --user "$(id -u):$(id -g)" \
            --name test_encrypted_publisher \
            -v $PWD/encryption/certificate-authority/ca-cert.pem:/mosquitto/encryption/certificate-authority/ca-cert.pem:ro \
            -v $PWD/encryption/client/client-cert.pem:/mosquitto/encryption/client/client-cert.pem:ro \
            -v $PWD/encryption/client/client-key.pem:/mosquitto/encryption/client/client-key.pem:ro \
            eclipse-mosquitto \
            mosquitto_pub \
                --cert /mosquitto/encryption/client/client-cert.pem \
                --key /mosquitto/encryption/client/client-key.pem \
                --cafile /mosquitto/encryption/certificate-authority/ca-cert.pem \
                -t "ping-encrypted" \
                -h localhost \
                --repeat 20 \
                --repeat-delay 1 \
                -p 8883 \
                -m "test_encrypted_connection_message"
        ```

        - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines and are running the above command on a different machine than the broker, you should not use `localhost` in the command above, but the Host + Domain Name, or the IP of the machine running the broker. It must be the same as what you entered for the Common Name when you created the certificates. You should clone this repository on the other machine and must place the already created ca certificate (*ca-cert.pem*) into [encryption/certificate-authority](encryption/certificate-authority) and the client certificate (*client-cert.pem*) and the client private key (*client-key.pem*) into [encryption/client](encryption/client).

    3. Start the subscriber in a new terminal.

        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --user "$(id -u):$(id -g)" \
            --name test_encrypted_subscriber \
            -v $PWD/encryption/certificate-authority/ca-cert.pem:/mosquitto/encryption/certificate-authority/ca-cert.pem:ro \
            -v $PWD/encryption/client/client-cert.pem:/mosquitto/encryption/client/client-cert.pem:ro \
            -v $PWD/encryption/client/client-key.pem:/mosquitto/encryption/client/client-key.pem:ro \
            eclipse-mosquitto \
            mosquitto_sub \
                --cert /mosquitto/encryption/client/client-cert.pem \
                --key /mosquitto/encryption/client/client-key.pem \
                --cafile /mosquitto/encryption/certificate-authority/ca-cert.pem \
                -t "ping-encrypted" \
                -h localhost
        ```

        - You should now see the `test_encrypted_connection_message` in the terminal of the subscriber.
        - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines and are running the above command on a different machine than the broker, you should not use `localhost` in the command above, but the Host + Domain Name, or the IP of the machine running the broker. It must be the same as what you entered for the Common Name when you created the certificates. You should clone this repository on the other machine and must place the already created ca certificate (*ca-cert.pem*) into [encryption/certificate-authority](encryption/certificate-authority) and the client certificate (*client-cert.pem*) and the client private key (*client-key.pem*) into [encryption/client](encryption/client).
    </details>

    <details>
    <summary>c) mosquitto_authenticated.ssl-server.conf</summary>
    
    1. Start the broker.
   
        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --name mosquitto-encrypted \
            -v $PWD/configs/mosquitto_authenticated.ssl-server.conf:/mosquitto/config/mosquitto.conf:ro \
            -v $PWD/authentication/mosquitto.passwd:/mosquitto/config/mosquitto.passwd:ro \
            -v $PWD/encryption/server:/mosquitto/encryption/server:ro \
            -v $PWD/encryption/certificate-authority:/mosquitto/encryption/certificate-authority:ro \
            --user "$(id -u):$(id -g)" \
            eclipse-mosquitto
        ```

    2. Start the publisher in a new terminal.
   
        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --user "$(id -u):$(id -g)" \
            --name test_encrypted_publisher \
            -v $PWD/encryption/certificate-authority/ca-cert.pem:/mosquitto/encryption/certificate-authority/ca-cert.pem:ro \
            -v $PWD/encryption/client/client-cert.pem:/mosquitto/encryption/client/client-cert.pem:ro \
            -v $PWD/encryption/client/client-key.pem:/mosquitto/encryption/client/client-key.pem:ro \
            eclipse-mosquitto \
            mosquitto_pub \
                --cafile /mosquitto/encryption/certificate-authority/ca-cert.pem \
                -u test_connection \
                -P test_connection \
                -t "ping-encrypted" \
                -h localhost \
                --repeat 20 \
                --repeat-delay 1 \
                -p 8883 \
                -m "test_encrypted_connection_message"
        ```

        - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines and are running the above command on a different machine than the broker, you should not use `localhost` in the command above, but the Host + Domain Name, or the IP of the machine running the broker. It must be the same as what you entered for the Common Name when you created the certificates. You should clone this repository on the other machine and must place the already created ca certificate (*ca-cert.pem*) into [encryption/certificate-authority](encryption/certificate-authority)

    3.  Start the subscriber in a new terminal.

        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --user "$(id -u):$(id -g)" \
            --name test_encrypted_subscriber \
            -v $PWD/encryption/certificate-authority/ca-cert.pem:/mosquitto/encryption/certificate-authority/ca-cert.pem:ro \
            -v $PWD/encryption/client/client-cert.pem:/mosquitto/encryption/client/client-cert.pem:ro \
            -v $PWD/encryption/client/client-key.pem:/mosquitto/encryption/client/client-key.pem:ro \
            eclipse-mosquitto \
            mosquitto_sub \
                --cafile /mosquitto/encryption/certificate-authority/ca-cert.pem \
                -u test_connection \
                -P test_connection \
                -t "ping-encrypted" \
                -h localhost
        ```

        - You should now see the `test_encrypted_connection_message` in the terminal of the subscriber.
        - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines and are running the above command on a different machine than the broker, you should not use `localhost` in the command above, but the Host + Domain Name, or the IP of the machine running the broker. It must be the same as what you entered for the Common Name when you created the certificates. You should clone this repository on the other machine and must place the already created ca certificate (*ca-cert.pem*) into [encryption/certificate-authority](encryption/certificate-authority)
    </details>

    <details>
    <summary>d) mosquitto_authenticated.ssl-all.conf </summary>
    
    1. Start the broker.
        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --name mosquitto-encrypted \
            -v $PWD/configs/mosquitto_authenticated.ssl-all.conf:/mosquitto/config/mosquitto.conf:ro \
            -v $PWD/authentication/mosquitto.passwd:/mosquitto/config/mosquitto.passwd:ro \
            -v $PWD/encryption/server:/mosquitto/encryption/server:ro \
            -v $PWD/encryption/certificate-authority:/mosquitto/encryption/certificate-authority:ro \
            --user "$(id -u):$(id -g)" \
            eclipse-mosquitto
        ```

    2. Start the publisher in a new terminal.

        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --user "$(id -u):$(id -g)" \
            --name test_encrypted_publisher \
            -v $PWD/encryption/certificate-authority/ca-cert.pem:/mosquitto/encryption/certificate-authority/ca-cert.pem:ro \
            -v $PWD/encryption/client/client-cert.pem:/mosquitto/encryption/client/client-cert.pem:ro \
            -v $PWD/encryption/client/client-key.pem:/mosquitto/encryption/client/client-key.pem:ro \
            eclipse-mosquitto \
            mosquitto_pub \
                --cert /mosquitto/encryption/client/client-cert.pem \
                --key /mosquitto/encryption/client/client-key.pem \
                --cafile /mosquitto/encryption/certificate-authority/ca-cert.pem \
                -u test_connection \
                -P test_connection \
                -t "ping-encrypted" \
                -h localhost \
                --repeat 20 \
                --repeat-delay 1 \
                -p 8883 \
                -m "test_encrypted_connection_message"
        ```

        - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines and are running the above command on a different machine than the broker, you should not use `localhost` in the command above, but the Host + Domain Name, or the IP of the machine running the broker. It must be the same as what you entered for the Common Name when you created the certificates. You should clone this repository on the other machine and must place the already created ca certificate (*ca-cert.pem*) into [encryption/certificate-authority](encryption/certificate-authority) and the client certificate (*client-cert.pem*) and the client private key (*client-key.pem*) into [encryption/client](encryption/client).
    
    3. Start the subscriber in a new terminal.

        ```bash
        # mqtt-broker $
        docker run \
            --rm \
            --network host \
            --user "$(id -u):$(id -g)" \
            --name test_encrypted_subscriber \
            -v $PWD/encryption/certificate-authority/ca-cert.pem:/mosquitto/encryption/certificate-authority/ca-cert.pem:ro \
            -v $PWD/encryption/client/client-cert.pem:/mosquitto/encryption/client/client-cert.pem:ro \
            -v $PWD/encryption/client/client-key.pem:/mosquitto/encryption/client/client-key.pem:ro \
            eclipse-mosquitto \
            mosquitto_sub \
                --cert /mosquitto/encryption/client/client-cert.pem \
                --key /mosquitto/encryption/client/client-key.pem \
                --cafile /mosquitto/encryption/certificate-authority/ca-cert.pem \
                -u test_connection \
                -P test_connection \
                -t "ping-encrypted" \
                -h localhost
        ```

        - You should now see the `test_encrypted_connection_message` in the terminal of the subscriber.
        - **Multi-Machine Setup**: If you want to test the encrypted setup across multiple machines and are running the above command on a different machine than the broker, you should not use `localhost` in the command above, but the Host + Domain Name, or the IP of the machine running the broker. It must be the same as what you entered for the Common Name when you created the certificates. You should clone this repository on the other machine and must place the already created ca certificate (*ca-cert.pem*) into [encryption/certificate-authority](encryption/certificate-authority) and the client certificate (*client-cert.pem*) and the client private key (*client-key.pem*) into [encryption/client](encryption/client).

    </details>

## Creating your own MQTT and Docker-based Applications

The instructions provided in this repository can only serve as the basis for developing your own systems which make use of Docker and MQTT.

While you may use existing broker implementation for most applications, an MQTT Client is often application-specific and may need to be implemented by yourself. Information on how to create a Python-based MQTT client can for example be found [here](https://github.com/eclipse/paho.mqtt.python).

If you are working in the field of robotics, we can recommend to use our software [mqtt_client](https://github.com/ika-rwth-aachen/mqtt_client), a generic ROS C++ Nodelet for bi-directionally bridging messages between ROS and MQTT. It is easy to integrate this software into any system based on the [Robot Operating System (ROS)](https://ros.org). It will help you connect multiple robots to each other or to external components, e.g. running in the cloud.

## Feedback

If these instructions helped you, feel free to give our repository a star ⭐. 

We are also happy about raised [issues](https://github.com/ika-rwth-aachen/mqtt-in-docker/issues), [pull requests](https://github.com/ika-rwth-aachen/mqtt-in-docker/pulls), and [discussions](https://github.com/ika-rwth-aachen/mqtt-in-docker/discussions).
