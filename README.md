# Generate Azure IoT Edge internal certificates using your own Root CA certificate

In this article we descibe a way to generate the intermediate and device certificate for use with an Azure IoT Edge for internal communication and as an transparant gateway, based on your own Root CA certificate and private key. You can use a Linux machine (Azure VM) to generate the certificates, and then copy them over to any IoT Edge device running on any supported operating system. This article uses openssl and the tools provided in the Azure IoT Edge github repository.

## The process

The process of creating the required IoT Edge internal certificates and keys as described in this article consists of a few steps:
1. Create a working directory on the development machine
2. Copy your Root CA key and certificate to the development machine
3. Clone the tools from the Azure IoT Edge repository and update theses tools for your use.
4. Create the intermediate certificate. You can create as nmany intermediates as you deem necessary by creating multiple working directories. These intermediates can be used to group IoT Edge devices. Some examples of intermediates can be: 'test', 'prod', etc.
2. Create the IoT Edge certificate and key for that intermediate certificate. These certificates are stored in the same location as the intermediate certificate they belong to.
3. Update the IoT Edge device

<p style="align:center">
<img src="images/Process.png">
</p>

## Prerequisites

* A development machine to create certificates. You can use an Ubuntu 18.04 Azure VM.
* A Root CA certificate and key file, that can generate an intermediate or leaf certificate. Your Root CA private key needs to be a RSA private key, 4096 bit long modulus and you need to know the password of the Root CA certificate.
* One or more Azure IoT Edge devices. Use the IoT Edge installation steps for one of the following operating systems:
  * [Windows](how-to-install-iot-edge-windows.md)
  * [Linux](how-to-install-iot-edge-linux.md)

## Prepare creation scripts

The Azure IoT Edge git repository contains scripts that you can use to generate the certificates. In this section, you clone the IoT Edge repo and execute the scripts. 

1. Login to your Linux development machine. Clone the git repo that contains scripts to generate (non-production) certificates. These scripts help you create the necessary certificates to set up an IoT Edge devices. 

   ```bash
   git clone https://github.com/Azure/iotedge.git
   ```

2. Navigate to the directory in which you want to work. We'll refer to this directory throughout the article as *\<WRKDIR>*. All certificate and key files will be created in this directory.
  
3. Copy the config and script files from the cloned IoT Edge repo into your working directory.

   ```bash
   cp <path>/iotedge/tools/CACertificates/*.cnf .
   cp <path>/iotedge/tools/CACertificates/certGen.sh .
   ```

4. Update certGen.sh to reflect your situation. Open the file `sudo nano certGen.sh`.

   Replace the following lines:
   ```yaml
   INTERMEDIATE_CA_PREFIX="azure-iot-test-only.intermediate"
   INTERMEDIATE_CA_PASSWORD="1234"
   ```

   with:
   ```yaml
   INTERMEDIATE_CA_PREFIX="<your intermediate prefix>"
   INTERMEDIATE_CA_PASSWORD="<your intermediate password>"
   ```

   Save the file.

## Create certificates

In this section, you create the intermediate and leaf certificates and then connect them in a chain. Placing the certificates in a chain file allows to easily install them on your IoT Edge gateway device and any downstream devices.

1. Create a directory to hold your Root CA certificate and key file. For instance *\<WRKDIR>/root*. Copy your Root CA certificate and key file to this directory.

2. Create the intermediate certificate. These certificates are placed in *\<WRKDIR>*.

   If you've already created root and intermediate certificates in this directory, don't run this script again. Rerunning this script will overwrite the existing certificates. Instead, proceed to the next step. 

   ```bash
   ../certGen.sh install_root_ca_from_files <path to your root certificate> <path to your root private key> <your private key password>
   ```

   The script creates the intermediate certificates and keys.

   > [!NB]
   > You can ignore the notification 'not for production' as your are using your own Root CA certificate and key.
   
3. Create the IoT Edge device CA certificate and private key with the following command. Provide a name for the CA certificate, for example the hostname of your IoT Edge device. The name is used to name the files and during certificate generation. 

   ```bash
   ../certGen.sh create_edge_device_ca_certificate "<your_iot_edge_hostname>"
   ```

   The script creates several certificates and keys. Make note of two, which we'll refer to in the next section: 
   * `<WRKDIR>/certs/iot-edge-device-ca-<your_iot_edge_hostname>-full-chain.cert.pem`
   * `<WRKDIR>/private/iot-edge-device-ca-<your_iot_edge_hostname>.key.pem`

   > [!NB]
   > You can ingore the notification 'not for production' warning as your are using your own Root CA certificate and key.

>[!TIP]
>If you provide a name for each IoT Edge device you have, then the certificates and keys created by this command will reflect that name. You can use these files to provide each IoT Edge with its own set of certificates.

>[!TIP]
>If you want to group IoT Edges based on intermediate certificates, you can just create a new working directory and go through these steps again for all IOT Edges in the specific group.

# Install certificates on the gateway

Now that you've made a certificate chain, you need to install it on the IoT Edge gateway device and configure the IoT Edge runtime to reference the new certificates. 

1. Copy the following files from *\<WRKDIR>/<intermediate_dir>/*. Save them anywhere on your IoT Edge device. We'll refer to the destination directory on your IoT Edge device as *\<CERTDIR>*. 

   * Device CA certificate -  `<WRKDIR>/certs/iot-edge-device-ca-<your_iot_edge_hostname>-full-chain.cert.pem`
   * Device CA private key - `<WRKDIR>/private/iot-edge-device-ca-<your_iot_edge_hostname>.key.pem`
   * Root CA - `<your root CA certificate file>`

   You can use a service like [Azure Key Vault](https://docs.microsoft.com/azure/key-vault) or a function like [Secure copy protocol](https://www.ssh.com/ssh/scp/) to move the certificate files.

2. Open the IoT Edge security daemon config file. 

   * Windows: `C:\ProgramData\iotedge\config.yaml`
   * Linux: `/etc/iotedge/config.yaml`

3. Set the **certificate** properties in the config.yaml file to the full path to the certificate and key files on the IoT Edge device. Remove the `#` character before the certificate properties to uncomment the four lines. Remember that indents in yaml are two spaces.

   * Windows:

      ```yaml
      certificates:
        device_ca_cert: "<CERTDIR>\\iot-edge-device-ca-<your_iot_edge_hostname>-full-chain.cert.pem"
        device_ca_pk: "<CERTDIR>\\iot-edge-device-ca-<your_iot_edge_hostname>.key.pem"
        trusted_ca_certs: "<CERTDIR>\\<your root CA certificate file>"
      ```
   
   * Linux: 
      ```yaml
      certificates:
        device_ca_cert: "<CERTDIR>/iot-edge-device-ca-<your_iot_edge_hostname>-full-chain.cert.pem"
        device_ca_pk: "<CERTDIR>/iot-edge-device-ca-<your_iot_edge_hostname>.key.pem"
        trusted_ca_certs: "<CERTDIR>/<your root CA certificate file>"
      ```

4. On Linux devices, make sure that the user **iotedge** has read permissions for the directory holding the certificates. 

# Certificate expiration on the gateway
The expiration for these certificates is set to 30 days, so you need to make sure you have a process in place to create new certificates for IoT Edge devices and roll them on the actuel IoT Edge device. You can use the steps above to partly automate the creation of new certificates every 30 days.

# Disclaimer

Be aware that you are using your own Root CA certificate and key and therefore must make sure you do this in a secured manner. This article is provided under the MIT license.