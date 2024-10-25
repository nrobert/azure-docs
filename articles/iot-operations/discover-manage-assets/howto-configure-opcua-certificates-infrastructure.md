---
title: Configure OPC UA certificates
description:  How to configure and manage the OPC UA certificates trust relationship in the context of the connector for OPC UA
author: dominicbetts
ms.author: dobett
ms.subservice: azure-opcua-connector
ms.custom: devx-track-azurecli
ms.topic: how-to
ms.date: 09/27/2024

# CustomerIntent: As an industrial edge IT or operations user, I want to to understand how to manage the OPC UA Certificates in the context of the connector for OPC UA.
---

# Configure OPC UA certificates infrastructure for the connector for OPC UA

[!INCLUDE [public-preview-note](../includes/public-preview-note.md)]

In this article, you learn how to configure the OPC UA certificates infrastructure for the connector for OPC UA. This configuration lets you determine which OPC UA servers you trust to securely establish a session with.

Based on the [OPC UA specification](https://reference.opcfoundation.org/), the connector for OPC UA acts as a single OPC UA application when it establishes secure communications with OPC UA servers. The connector for OPC UA uses the same application instance certificate for all secure channels it opens to your OPC UA servers.

To learn more, see [OPC UA certificates infrastructure for the connector for OPC UA](overview-opcua-broker-certificates-management.md).

## Prerequisites

A deployed instance of Azure IoT Operations Preview. To deploy Azure IoT Operations for demonstration and exploration purposes, see [Quickstart: Run Azure IoT Operations Preview in GitHub Codespaces with K3s](../get-started-end-to-end-sample/quickstart-deploy.md).

## Configure a self-signed application instance certificate

The default deployment of the connector for OPC UA installs all the resources needed by [cert-manager](https://cert-manager.io/) to create an OPC UA compliant certificate. A self-signed CA is used to sign this certificate. The application instance certificate is stored in the `aio-opc-opcuabroker-default-application-cert` secret while the CA certificate is stored in `aio-opc-opcuabroker-default-root-ca-cert` secret. The `aio-opc-opcuabroker-default-application-cert` secret is mapped into all the connector for OPC UA pods and acts as the OPC UA client application instance certificate. `cert-manager` handles the automatic renewal of both the application instance certificate and the self signed CA.

This configuration is typically sufficient for compliant and secure communication between your OPC UA servers and the connector for OPC UA in a demonstration or exploration environment. For a production environment, use enterprise grade application instance certificates in your deployment.

## Configure the trusted certificates list

To connect to an asset, first you need to establish the application authentication mutual trust. For the connector for OPC UA, complete the following steps:

1. Get the OPC UA server application's instance certificate as a file. These files typically have a .der or .crt extension. This is the public key only.

    > [!TIP]
    > Typically, an OPC UA server has an interface that lets you export its application instance certificate. This interface isn't standardized. For servers such as KEPServerEx, there's a Windows-based configuration UI for certificates management. Other servers might have a web interface or use operating system folders to store the certificates. Refer to the user manual of your server to find out how to export the application instance certificate. After you have the certificate, make sure it's either DER or PEM encoded. Typically stored in files with either the .der or .crt extension. If the certificate isn't in one of those file formats, use a tool such as `openssl` to transform the certificate into the required format.

1. Add the OPC UA server's application instance certificate to the trusted certificates list. This list is implemented as a Kubernetes native secret named *aio-opc-ua-broker-trust-list* that's created when you deploy Azure IoT Operations.

    # [Bash](#tab/bash)

    For a DER encoded certificate in a file such as *./my-server.der*, run the following command:

    ```bash
    # Append my-server.der OPC UA server certificate to the trusted certificate list secret as a new entry
    data=$(kubectl create secret generic temp --from-file=my-server.der=./my-server.der --dry-run=client -o jsonpath='{.data}')
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{\"data\": $data}"
    ```

    For a PEM encoded certificate in a file such as *./my-server.crt*, run the following command:

    ```bash
    # Append my-server.crt OPC UA server certificate to the trusted certificate list secret as a new entry
    data=$(kubectl create secret generic temp --from-file=my-server.crt=./my-server.crt --dry-run=client -o jsonpath='{.data}')
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{\"data\": $data}"
    ```

    # [PowerShell](#tab/powershell)

    For a DER encoded certificate in a file such as *./my-server.der*, run the following command:

    ```powershell
    # Append my-server.der OPC UA server certificate to the trusted certificate list secret as a new entry
    $data = kubectl create secret generic temp --from-file=my-server.der=./my-server.der --dry-run=client -o jsonpath='{.data}'
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"
    ```

    For a PEM encoded certificate in a file such as *./my-server.crt*, run the following command:

    ```powershell
    # Append my-server.crt OPC UA server certificate to the trusted certificate list secret as a new entry
    $data = kubectl create secret generic temp --from-file=my-server.crt=./my-server.crt --dry-run=client -o jsonpath='{.data}'
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"
    ```

    ---

If your OPC UA server uses a certificate issued by a certificate authority (CA), you can trust the CA by adding its public key certificate to the connector for OPC UA trusted certificates list. The connector for OPC UA now automatically trusts all the servers that use a valid certificate issued by the CA. Therefore, you don't need to explicitly add the OPC UA server's certificate to the connector for OPC UA trusted certificates list.

To trust a CA, complete the following steps:

1. Get the CA certificate public key encode in DER or PEM format. These certificates are typically stored in files with either the .der or .crt extension. Get the CA's CRL. This list is typically in a file with the .crl. Check the documentation for your OPC UA server for details.

1. Save the CA certificate and the CRL in the *aio-opc-ua-broker-trust-list* Kubernetes native secret.

    # [Bash](#tab/bash)

    For a DER encoded CA certificate in a file such as *./my-server-ca.der*, run the following commands:

    ```bash
    # Append CA certificate to the trusted certificate list secret as a new entry
    data=$(kubectl create secret generic temp --from-file=my-server-ca.der=./my-server-ca.der --dry-run=client -o jsonpath='{.data}')
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"

    # Append the CRL to the trusted certificate list secret as a new entry
    data=$(kubectl create secret generic temp --from-file= my-server-ca.crl=./ my-server-ca.crl --dry-run=client -o jsonpath='{.data}')
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"
    ```

    For a PEM encoded CA certificate in a file such as *./my-server-ca.crt*, run the following commands:

    ```bash
    # Append CA certificate to the trusted certificate list secret as a new entry
    data=$(kubectl create secret generic temp --from-file=my-server-ca.crt=./my-server-ca.crt --dry-run=client -o jsonpath='{.data}')
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"

    # Append the CRL to the trusted certificates list secret as a new entry
    data=$(kubectl create secret generic temp --from-file=my-server-ca.crl=./my-server-ca.crl --dry-run=client -o jsonpath='{.data}')
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"
    ```

    # [PowerShell](#tab/powershell)

    For a DER encoded certificate in a file such as *./my-server-ca.der*, run the following commands:

    ```powershell
    # Append CA certificate to the trusted certificate list secret as a new entry
    $data = kubectl create secret generic temp --from-file=my-server-ca.der=./my-server-ca.der --dry-run=client -o jsonpath='{.data}'
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"

    # Append the CRL to the trusted certificate list secret as a new entry
    $data = kubectl create secret generic temp --from-file=my-server-ca.crl=./my-server-ca.crl --dry-run=client -o jsonpath='{.data}'
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"
    ```

    For a PEM encoded certificate in a file such as *./my-server-ca.crt*, run the following commands:

    ```powershell
    # Append CA certificate to the trusted certificate list secret as a new entry
    $data = kubectl create secret generic temp --from-file=my-server-ca.crt=./my-server-ca.crt --dry-run=client -o jsonpath='{.data}'
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"

    # Append the CRL to the trusted certificate list secret as a new entry
    $data = kubectl create secret generic temp --from-file=my-server-ca.crl=./my-server-ca.crl --dry-run=client -o jsonpath='{.data}'
    kubectl patch secret aio-opc-ua-broker-trust-list -n azure-iot-operations -p "{`"data`": $data}"
    ```

    ---

## Configure the issuer certificates list

If your OPC UA server uses a certificate issued by a CA, but you don't want to trust all certificates issued by the CA, complete the following steps:

1. Trust the OPC UA server's application instance certificate by following the first three steps in the previous section.

1. Besides the certificate itself, the connector for OPC UA needs the CA certificate to properly validate the issuer chain of the OPC UA server's certificate. Add the CA certificate and its certificate revocation list (CRL) to a separate list called *aio-opc-ua-broker-issuer-list* that's implemented as a Kubernetes secret.

    1. Save the CA certificate and the CRL in the `aio-opc-ua-broker-issuer-list` secret.

        # [Bash](#tab/bash)

        For a DER encoded certificate in a file such as *./my-server-ca.der*, run the following commands:

        ```bash
        # Append CA certificate to the issuer list secret as a new entry
        data=$(kubectl create secret generic temp --from-file=my-server-ca.der=./my-server-ca.der --dry-run=client -o jsonpath='{.data}')
        kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"

        # Append the CRL to the issuer list secret as a new entry
        data=$(kubectl create secret generic temp --from-file=my-server-ca.crl=./my-server-ca.crl --dry-run=client -o jsonpath='{.data}')
        kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"
        ```

        For a PEM encoded certificate in a file such as *./my-server-ca.crt*, run the following commands:

        ```bash
        # Append CA certificate to the issuer list secret as a new entry
        data=$(kubectl create secret generic temp --from-file=my-server-ca.crt=./my-server-ca.crt --dry-run=client -o jsonpath='{.data}')
        kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"

        # Append the CRL to the issuer list secret as a new entry
        data=$(kubectl create secret generic temp --from-file=my-server-ca.crl=./my-server-ca.crl --dry-run=client -o jsonpath='{.data}')
        kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"
        ```

        # [PowerShell](#tab/powershell)

        For a DER encoded certificate in a file such as *./my-server-ca.der*, run the following commands:

        ```powershell
        # Append CA certificate to the issuer list secret as a new entry
        $data = kubectl create secret generic temp --from-file=my-server-ca.der=./my-server-ca.der --dry-run=client -o jsonpath='{.data}'
        kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"

        # Append the CRL to the issuer list secret as a new entry
        $data = kubectl create secret generic temp --from-file=my-server-ca.crl=./my-server-ca.crl --dry-run=client -o jsonpath='{.data}'
        kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"
        ```

        For a PEM encoded certificate in a file such as *./my-server-ca.crt*, run the following commands:

        ```powershell
        # Append CA certificate to the issuer list secret as a new entry
        $data = kubectl create secret generic temp --from-file=my-server-ca.crt=./my-server-ca.crt --dry-run=client -o jsonpath='{.data}'
        kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"

        # Append the CRL to the issuer list secret as a new entry
        $data = kubectl create secret generic temp --from-file=my-server-ca.crl=./my-server-ca.crl --dry-run=client -o jsonpath='{.data}'
        kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"
        ```

        ---

## Configure your OPC UA server

To complete the configuration of the application authentication mutual trust, you need to configure your OPC UA server to trust the connector for OPC UA application instance certificate together with its issuer trust chain:

1. To extract the public key certificate for the OPC UA connector into a `opcuabroker.crt` file, run the following command:

    # [Bash](#tab/bash)

    ```bash
    kubectl -n azure-iot-operations get secret aio-opc-opcuabroker-default-application-cert -o jsonpath='{.data.tls\.crt}' | base64 -d > opcuabroker.crt
    ```

    # [PowerShell](#tab/powershell)

    ```powershell
    kubectl -n azure-iot-operations get secret aio-opc-opcuabroker-default-application-cert -o jsonpath='{.data.tls\.crt}' | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) } > opcuabroker.crt
    ```

    ---

1. To extract the CA public key certificate for the OPC UA connector into a `opcuabroker-ca.crt` file, run the following command:

    # [Bash](#tab/bash)

    ```bash
    kubectl -n azure-iot-operations get secret aio-opc-opcuabroker-default-application-cert -o jsonpath='{.data.ca\.crt}' | base64 -d > opcuabroker-ca.crt
    ```

    # [PowerShell](#tab/powershell)

    ```powershell
    kubectl -n azure-iot-operations get secret aio-opc-opcuabroker-default-application-cert -o jsonpath='{.data.ca\.crt}' | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) } > opcuabroker-ca.crt
    ```

    ---

1. Many OPC UA servers only support certificates in the DER format. If necessary, use the following command to convert the _opcuabroker.crt_ and _opcuabroker-ca.crt_ certificates to _opcuabroker.der_ and _opcuabroker-ca.der_:

    ```bash
    openssl x509 -outform der -in opcuabroker.crt -out opcuabroker.der
    openssl x509 -outform der -in opcuabroker-ca.crt -out opcuabroker-ca.der
    ```

1. Consult the documentation of your OPC UA server to learn how to add the `opcuabroker.crt` or `opcuabroker.der` certificate file to the server's trusted certificates list, and the `opcuabroker-ca.crt` or `opcuabroker-ca.der` CA certificate file into the server's trusted issuers list.

## Configure an enterprise grade application instance certificate

For production environments, you can configure the connector for OPC UA to use an enterprise grade application instance certificate. Typically, an enterprise CA issues this certificate and you need the CA certificate to your configuration. Often, there's a hierarchy of CAs and you need to add the complete validation chain of CAs to your configuration.

The following example references the following items:

| Item | Description |
| ---- | ----------- |
| _opcuabroker-certificate.der_ | File that contains the enterprise grade application instance certificate public key. |
| _opcuabroker-certificate.pem_ | File that contains the enterprise grade application instance certificate private key. |
| `subjectName`                 | The subject name string embedded in the application instance certificate. |
| `applicationUri`              | The application instance URI embedded in the application instance. |
| _enterprise-grade-ca-1.der_   | File that contains the enterprise grade CA certificate public key. |
| _enterprise-grade-ca-1.crl_   | The CA's CRL file. |

Like the previous examples, you use a dedicated Kubernetes secret to store the certificates and CRLs. To configure the enterprise grade application instance certificate, complete the following steps:

1. Save the certificates and the CRL in the *aio-opc-ua-broker-client-certificate* secret by using the following command:

    # [Bash](#tab/bash)

    ```bash
    # Create aio-opc-ua-broker-client-certificate secret
    # Upload OPC UA public key certificate as an entry to the secret
    # Upload OPC UA private key certificate as an entry to the secret
    kubectl create secret generic aio-opc-ua-broker-client-certificate -n azure-iot-operations  \
      --from-file=opcuabroker-certificate.der=./opcuabroker-certificate.der \
      --from-file=opcuabroker-certificate.pem=./opcuabroker-certificate.pem
    ```

    # [PowerShell](#tab/powershell)

    ```powershell
    # Create aio-opc-ua-broker-client-certificate secret
    # Upload OPC UA public key certificate as an entry to the secret
    # Upload OPC UA private key certificate as an entry to the secret
    kubectl create secret generic aio-opc-ua-broker-client-certificate -n azure-iot-operations  `
      --from-file=opcuabroker-certificate.der=./opcuabroker-certificate.der `
      --from-file=opcuabroker-certificate.pem=./opcuabroker-certificate.pem
    ```

    ---

2. If you use the CA to issue certificates for your OPC UA broker, configure the *aio-opc-ua-broker-issuer-list* secret. Use a Kubernetes client such as `kubectl` to configure the secrets *enterprise-grade-ca-1.der* and *enterprise-grade-ca-1.crl*:

    # [Bash](#tab/bash)


    ```bash
    # Append CA certificate to the issuer list secret as a new entry
    data=$(kubectl create secret generic temp --from-file=enterprise-grade-ca-1.der=./enterprise-grade-ca-1.der --dry-run=client -o jsonpath='{.data}')
    kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"

    # Append the CRL to the issuer list secret as a new entry
    data=$(kubectl create secret generic temp --from-file= enterprise-grade-ca-1.crl=./enterprise-grade-ca-1.crl --dry-run=client -o jsonpath='{.data}')
    kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"
    ```

    # [PowerShell](#tab/powershell)

    ```powershell
    # Append CA certificate to the issuer list secret as a new entry
    $data = kubectl create secret generic temp --from-file=enterprise-grade-ca-1.der=./enterprise-grade-ca-1.der --dry-run=client -o jsonpath='{.data}'
    kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"

    # Append the CRL to the issuer list secret as a new entry
    $data = kubectl create secret generic temp --from-file=enterprise-grade-ca-1.crl=./enterprise-grade-ca-1.crl --dry-run=client -o jsonpath='{.data}'
    kubectl patch secret aio-opc-ua-broker-issuer-list -n azure-iot-operations -p "{`"data`": $data}"
    ```

    ---

1. Update the connector for OPC UA deployment to use the new `secret` source for application instance certificates by using the following command:

    # [Bash](#tab/bash)

    ```bash
   az k8s-extension update \
        --version 0.7.0-preview \
        --name azure-iot-operations-qlll2 \
        --release-train preview \
        --cluster-name <cluster-name> \
        --resource-group <azure-resource-group> \
        --cluster-type connectedClusters \
        --auto-upgrade-minor-version false \
        --config connectors.values.securityPki.applicationCert=aio-opc-ua-broker-client-certificate \
        --config connectors.values.securityPki.subjectName=<subjectName> \
        --config connectors.values.securityPki.applicationUri=<applicationUri>
    ```

    # [PowerShell](#tab/powershell)

    ```powershell
   az k8s-extension update `
        --version 0.7.0-preview `
        --name azure-iot-operations-qlll2 `
        --release-train preview `
        --cluster-name <cluster-name> `
        --resource-group <azure-resource-group> `
        --cluster-type connectedClusters `
        --auto-upgrade-minor-version false `
        --config connectors.values.securityPki.applicationCert=aio-opc-ua-broker-client-certificate `
        --config connectors.values.securityPki.subjectName=<subjectName> `
        --config connectors.values.securityPki.applicationUri=<applicationUri>
    ```

    ---

Now that the connector for OPC UA uses the enterprise certificate, don't forget to add the new certificate's public key to the trusted certificate lists of all OPC UA servers it needs to connect to.
