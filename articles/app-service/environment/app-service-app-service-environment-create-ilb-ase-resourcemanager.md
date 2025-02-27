---
title: Create ILB App Service Environment v1
description: Create an App Service environment (ASE) with an internal load balancer (ILB). This doc is provided only for customers who use the legacy v1 ASE.
author: madsd

ms.assetid: 091decb6-b0de-42a1-9f2f-c18d9b2e67df
ms.topic: article
ms.date: 03/15/2022
ms.author: madsd
ms.custom: seodec18

---
# How To Create an ILB ASEv1 Using Azure Resource Manager Templates

> [!IMPORTANT]
> This article is about App Service Environment v1. App Service Environment v1 will be retired on 31 August 2024. There's a new version of App Service Environment that is easier to use and runs on more powerful infrastructure. To learn more about the new version, start with the [Introduction to the App Service Environment](overview.md). If you're currently using App Service Environment v1, please follow the steps in [this article](migration-alternatives.md) to migrate to the new version.
>

## Overview
App Service Environments can be created with a virtual network internal address instead of a public VIP. This internal address is provided by an Azure component called the internal load balancer (ILB). An ILB ASE can be created using the Azure portal. It can also be created using automation by way of Azure Resource Manager templates. This article walks through the steps and syntax needed to create an ILB ASE with Azure Resource Manager templates.

There are three steps involved in automating creation of an ILB ASE:

1. First the base ASE is created in a virtual network using an internal load balancer address instead of a public VIP. As part of this step, a root domain name is assigned to the ILB ASE.
2. Once the ILB ASE is created, a TLS/SSL certificate is uploaded.
3. The uploaded TLS/SSL certificate is explicitly assigned to the ILB ASE as its "default" TLS/SSL certificate. This TLS/SSL certificate will be used for TLS traffic to apps on the ILB ASE when the apps are addressed using the common root domain assigned to the ASE (for example `https://someapp.mycustomrootcomain.com`)

## Creating the Base ILB ASE
An example Azure Resource Manager template, and its associated parameters file, are available [here][quickstartilbasecreate].

Most of the parameters in the *azuredeploy.parameters.json* file are common to creating both ILB ASEs, and ASEs bound to a public VIP. The list below calls out parameters of special note, or that are unique, when creating an ILB ASE:

* *internalLoadBalancingMode*:  Determines how control and data ports are exposed.
    * *3* means both HTTP/HTTPS traffic on ports 80/443, and the control/data channel ports listened to by the FTP service on the ASE, will be bound to an ILB allocated virtual network internal address.
    * *2* means only the FTP service related ports (both control and data channels) will be bound to an ILB address, while the HTTP/HTTPS traffic will remain on the public VIP.
    * *0* means all traffic is bound to the public VIP making the ASE external.
* *dnsSuffix*:  This parameter defines the default root domain that will be assigned to the ASE. In the public variation of Azure App Service, the default root domain for all web apps is *azurewebsites.net*. However since an ILB ASE is internal to a customer's virtual network, it doesn't make sense to use the public service's default root domain. Instead, an ILB ASE should have a default root domain that makes sense for use within a company's internal virtual network. For example, a hypothetical Contoso Corporation might use a default root domain of *internal-contoso.com* for apps that are intended to only be resolvable and accessible within Contoso's virtual network. 
* *ipSslAddressCount*:  This parameter is automatically defaulted to a value of 0 in the *azuredeploy.json* file because ILB ASEs only have a single ILB address. There are no explicit IP-SSL addresses for an ILB ASE, and so the IP-SSL address pool for an ILB ASE must be set to zero, otherwise a provisioning error will occur. 

Once the *azuredeploy.parameters.json* file has been filled in for an ILB ASE, the ILB ASE can then be created using the following PowerShell code snippet.  Change the file paths to match where the Azure Resource Manager template files are located on your machine.  Also remember to supply your own values for the Azure Resource Manager deployment name, and resource group name.

```azurepowershell-interactive
$templatePath="PATH\azuredeploy.json"
$parameterPath="PATH\azuredeploy.parameters.json"

New-AzResourceGroupDeployment -Name "CHANGEME" -ResourceGroupName "YOUR-RG-NAME-HERE" -TemplateFile $templatePath -TemplateParameterFile $parameterPath
```

After the Azure Resource Manager template is submitted, it will take a few hours for the ILB ASE to be created. Once the creation completes, the ILB ASE will show up in the portal UX in the list of App Service Environments for the subscription that triggered the deployment.

## Uploading and Configuring the "Default" TLS/SSL Certificate
Once the ILB ASE is created, a TLS/SSL certificate should be associated with the ASE as the "default" TLS/SSL certificate use for establishing TLS/SSL connections to apps.  Continuing with the hypothetical Contoso Corporation example, if the ASE's default DNS suffix is *internal-contoso.com*, then a connection to *`https://some-random-app.internal-contoso.com`* requires a TLS/SSL certificate that is valid for **.internal-contoso.com*. 

There are different ways to obtain a valid TLS/SSL certificate including internal CAs, purchasing a certificate from an external issuer, and using a self-signed certificate.  Regardless of the source of the TLS/SSL certificate, the following certificate attributes need to be configured properly:

* *Subject*:  This attribute must be set to **.your-root-domain-here.com*
* *Subject Alternative Name*:  This attribute must include both **.your-root-domain-here.com*, and **.scm.your-root-domain-here.com*.  The reason for the second entry is that TLS connections to the SCM/Kudu site associated with each app will be made using an address of the form *your-app-name.scm.your-root-domain-here.com*.

With a valid TLS/SSL certificate in hand, two additional preparatory steps are needed. The TLS/SSL certificate needs to be converted/saved as a .pfx file. Remember that the .pfx file needs to include all intermediate and root certificates, and also needs to be secured with a password.

Then the resultant .pfx file needs to be converted into a base64 string because the TLS/SSL certificate will be uploaded using an Azure Resource Manager template.  Since Azure Resource Manager templates are text files, the .pfx file needs to be converted into a base64 string so it can be included as a parameter of the template.

The PowerShell code snippet below shows an example of generating a self-signed certificate, exporting the certificate as a .pfx file, converting the .pfx file into a base64 encoded string, and then saving the base64 encoded string to a separate file. The PowerShell code for base64 encoding was adapted from the [PowerShell Scripts Blog][examplebase64encoding].

```azurepowershell-interactive
$certificate = New-SelfSignedCertificate -certstorelocation cert:\localmachine\my -dnsname "*.internal-contoso.com","*.scm.internal-contoso.com"

$certThumbprint = "cert:\localMachine\my\" + $certificate.Thumbprint
$password = ConvertTo-SecureString -String "CHANGETHISPASSWORD" -Force -AsPlainText

$fileName = "exportedcert.pfx"
Export-PfxCertificate -cert $certThumbprint -FilePath $fileName -Password $password     

$fileContentBytes = get-content -encoding byte $fileName
$fileContentEncoded = [System.Convert]::ToBase64String($fileContentBytes)
$fileContentEncoded | set-content ($fileName + ".b64")
```

Once the TLS/SSL certificate has been successfully generated and converted to a base64 encoded string, the example Azure Resource Manager template for [configuring the default TLS/SSL certificate][configuringDefaultSSLCertificate] can be used.

The parameters in the *azuredeploy.parameters.json* file are listed below:

* *appServiceEnvironmentName*:  The name of the ILB ASE being configured.
* *existingAseLocation*:  Text string containing the Azure region where the ILB ASE was deployed.  For example:  "South Central US".
* *pfxBlobString*:  The based64 encoded string representation of the .pfx file.  Using the code snippet shown earlier, you would copy the string contained in "exportedcert.pfx.b64" and paste it in as the value of the *pfxBlobString* attribute.
* *password*:  The password used to secure the .pfx file.
* *certificateThumbprint*:  The certificate's thumbprint.  If you retrieve this value from PowerShell (for example *$certThumbprint* from the earlier code snippet), you can use the value as-is.  However if you copy the value from the Windows certificate dialog, remember to strip out the extraneous spaces.  The *certificateThumbprint* should look something like:  AF3143EB61D43F6727842115BB7F17BBCECAECAE
* *certificateName*:  A friendly string identifier of your own choosing used to identity the certificate. The name is used as part of the unique Azure Resource Manager identifier for the *Microsoft.Web/certificates* entity representing the TLS/SSL certificate. The name **must** end with the following suffix: \_yourASENameHere_InternalLoadBalancingASE. This suffix is used by the portal as an indicator that the certificate is used for securing an ILB-enabled ASE.

An abbreviated example of *azuredeploy.parameters.json* is shown below:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "appServiceEnvironmentName": {
            "value": "yourASENameHere"
        },
        "existingAseLocation": {
            "value": "East US 2"
        },
        "pfxBlobString": {
            "value": "MIIKcAIBAz...snip...snip...pkCAgfQ"
        },
        "password": {
            "value": "PASSWORDGOESHERE"
        },
        "certificateThumbprint": {
            "value": "AF3143EB61D43F6727842115BB7F17BBCECAECAE"
        },
        "certificateName": {
            "value": "DefaultCertificateFor_yourASENameHere_InternalLoadBalancingASE"
        }
    }
}
```

Once the *azuredeploy.parameters.json* file has been filled in, the default TLS/SSL certificate can be configured using the following PowerShell code snippet. Change the file paths to match where the Azure Resource Manager template files are located on your machine. Also remember to supply your own values for the Azure Resource Manager deployment name, and resource group name.

```azurepowershell-interactive
$templatePath="PATH\azuredeploy.json"
$parameterPath="PATH\azuredeploy.parameters.json"

New-AzResourceGroupDeployment -Name "CHANGEME" -ResourceGroupName "YOUR-RG-NAME-HERE" -TemplateFile $templatePath -TemplateParameterFile $parameterPath
```

After the Azure Resource Manager template is submitted, it will take roughly 40 minutes per ASE front-end to apply the change. For example, with a default sized ASE using two front-ends, the template will take around 1 hour and 20 minutes to complete. While the template is running the ASE will not be able to scaled.

Once the template completes, apps on the ILB ASE can be accessed over HTTPS and the connections will be secured using the default TLS/SSL certificate. The default TLS/SSL certificate will be used when apps on the ILB ASE are addressed using a combination of the application name plus the default hostname.  For example, *`https://mycustomapp.internal-contoso.com`* would use the default TLS/SSL certificate for **.internal-contoso.com*.

However, just like apps running on the public multi-tenant service, developers can also configure custom host names for individual apps, and then configure unique SNI TLS/SSL certificate bindings for individual apps.

## Getting started
To get started with App Service Environments, see [Introduction to App Service Environment](app-service-app-service-environment-intro.md)

[!INCLUDE [app-service-web-try-app-service](../../../includes/app-service-web-try-app-service.md)]

<!-- LINKS -->
[quickstartilbasecreate]: https://azure.microsoft.com/resources/templates/web-app-ase-ilb-create/
[examplebase64encoding]: https://powershellscripts.blogspot.com/2007/02/base64-encode-file.html 
[configuringDefaultSSLCertificate]: https://azure.microsoft.com/resources/templates/web-app-ase-ilb-configure-default-ssl/

