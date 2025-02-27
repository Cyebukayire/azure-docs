---
title: Prepare to deploy your solution in production - Azure IoT Edge
description: Learn how to take your Azure IoT Edge solution from development to production, including setting up your devices with the appropriate certificates and making a deployment plan for future code updates. 
author: PatAltimore

ms.author: patricka
ms.date: 03/01/2021
ms.topic: conceptual
ms.service: iot-edge
services: iot-edge
ms.custom:  [amqp, mqtt]
---

# Prepare to deploy your IoT Edge solution in production

[!INCLUDE [iot-edge-version-201806-or-202011](../../includes/iot-edge-version-201806-or-202011.md)]

When you're ready to take your IoT Edge solution from development into production, make sure that it's configured for ongoing performance.

The information provided in this article isn't all equal. To help you prioritize, each section starts with lists that divide the work into two sections: **important** to complete before going to production, or **helpful** for you to know.

## Device configuration

IoT Edge devices can be anything from a Raspberry Pi to a laptop to a virtual machine running on a server. You may have access to the device either physically or through a virtual connection, or it may be isolated for extended periods of time. Either way, you want to make sure that it's configured to work appropriately.

* **Important**
  * Install production certificates
  * Have a device management plan
  * Use Moby as the container engine

* **Helpful**
  * Choose upstream protocol

### Install production certificates

Every IoT Edge device in production needs a device certificate authority (CA) certificate installed on it. That CA certificate is then declared to the IoT Edge runtime in the config file. For development and testing scenarios, the IoT Edge runtime creates temporary certificates if no certificates are declared in the config file. However, these temporary certificates expire after three months and aren't secure for production scenarios. For production scenarios, you should provide your own device CA certificate, either from a self-signed certificate authority or purchased from a commercial certificate authority.

<!--1.1-->
:::moniker range="iotedge-2018-06"

> [!NOTE]
> Currently, a limitation in libiothsm prevents the use of certificates that expire on or after January 1, 2038.

:::moniker-end

To understand the role of the device CA certificate, see [How Azure IoT Edge uses certificates](iot-edge-certs.md).

For more information about how to install certificates on an IoT Edge device and reference them from the config file, see [Manage certificate on an IoT Edge device](how-to-manage-device-certificates.md).

### Have a device management plan

Before you put any device in production you should know how you're going to manage future updates. For an IoT Edge device, the list of components to update may include:

* Device firmware
* Operating system libraries
* Container engine, like Moby
* IoT Edge
* CA certificates

For more information, see [Update the IoT Edge runtime](how-to-update-iot-edge.md). The current methods for updating IoT Edge require physical or SSH access to the IoT Edge device. If you have many devices to update, consider adding the update steps to a script or use an automation tool like Ansible.

### Use Moby as the container engine

A container engine is a prerequisite for any IoT Edge device. Only moby-engine is supported in production. Other container engines, like Docker, do work with IoT Edge and it's ok to use these engines for development. The moby-engine can be redistributed when used with Azure IoT Edge, and Microsoft provides servicing for this engine.

### Choose upstream protocol

You can configure the protocol (which determines the port used) for upstream communication to IoT Hub for both the IoT Edge agent and the IoT Edge hub. The default protocol is AMQP, but you may want to change that depending on your network setup.

The two runtime modules both have an **UpstreamProtocol** environment variable. The valid values for the variable are:

* MQTT
* AMQP
* MQTTWS
* AMQPWS

Configure the UpstreamProtocol variable for the IoT Edge agent in the config file on the device itself. For example, if your IoT Edge device is behind a proxy server that blocks AMQP ports, you may need to configure the IoT Edge agent to use AMQP over WebSocket (AMQPWS) to establish the initial connection to IoT Hub.

Once your IoT Edge device connects, be sure to continue configuring the UpstreamProtocol variable for both runtime modules in future deployments. An example of this process is provided in [Configure an IoT Edge device to communicate through a proxy server](how-to-configure-proxy-support.md).

## Deployment

* **Helpful**
  * Be consistent with upstream protocol
  * Set up host storage for system modules
  * Reduce memory space used by the IoT Edge hub
  * Do not use debug versions of module images
  * Be mindful of twin size limits when using custom modules

### Be consistent with upstream protocol

If you configured the IoT Edge agent on your IoT Edge device to use a different protocol than the default AMQP, then you should declare the same protocol in all future deployments. For example, if your IoT Edge device is behind a proxy server that blocks AMQP ports, you probably configured the device to connect over AMQP over WebSocket (AMQPWS). When you deploy modules to the device, configure the same AMQPWS protocol for the IoT Edge agent and IoT Edge hub, or else the default AMQP will override the settings and prevent you from connecting again.

You only have to configure the UpstreamProtocol environment variable for the IoT Edge agent and IoT Edge hub modules. Any additional modules adopt whatever protocol is set in the runtime modules.

An example of this process is provided in [Configure an IoT Edge device to communicate through a proxy server](how-to-configure-proxy-support.md).

### Set up host storage for system modules

The IoT Edge hub and agent modules use local storage to maintain state and enable messaging between modules, devices, and the cloud. For better reliability and performance, configure the system modules to use storage on the host filesystem.

For more information, see [Host storage for system modules](how-to-access-host-storage-from-module.md).

### Reduce memory space used by IoT Edge hub

If you're deploying constrained devices with limited memory available, you can configure IoT Edge hub to run in a more streamlined capacity and use less disk space. These configurations do limit the performance of the IoT Edge hub, however, so find the right balance that works for your solution.

#### Don't optimize for performance on constrained devices

The IoT Edge hub is optimized for performance by default, so it attempts to allocate large chunks of memory. This configuration can cause stability problems on smaller devices like the Raspberry Pi. If you're deploying devices with constrained resources, you may want to set the **OptimizeForPerformance** environment variable to **false** on the IoT Edge hub.

When **OptimizeForPerformance** is set to **true**, the MQTT protocol head uses the PooledByteBufferAllocator, which has better performance but allocates more memory. The allocator does not work well on 32-bit operating systems or on devices with low memory. Additionally, when optimized for performance, RocksDb allocates more memory for its role as the local storage provider.

For more information, see [Stability issues on smaller devices](troubleshoot-common-errors.md#stability-issues-on-smaller-devices).

#### Disable unused protocols

Another way to optimize the performance of the IoT Edge hub and reduce its memory usage is to turn off the protocol heads for any protocols that you're not using in your solution.

Protocol heads are configured by setting boolean environment variables for the IoT Edge hub module in your deployment manifests. The three variables are:

* **amqpSettings__enabled**
* **mqttSettings__enabled**
* **httpSettings__enabled**

All three variables have *two underscores* and can be set to either true or false.

#### Reduce storage time for messages

The IoT Edge hub module stores messages temporarily if they cannot be delivered to IoT Hub for any reason. You can configure how long the IoT Edge hub holds on to undelivered messages before letting them expire. If you have memory concerns on your device, you can lower the **timeToLiveSecs** value in the IoT Edge hub module twin.

The default value of the timeToLiveSecs parameter is 7200 seconds, which is two hours.

### Do not use debug versions of module images

When moving from test scenarios to production scenarios, remember to remove debug configurations from deployment manifests. Check that none of the module images in the deployment manifests have the **\.debug** suffix. If you added create options to expose ports in the modules for debugging, remove those create options as well.

### Be mindful of twin size limits when using custom modules

The deployment manifest that contains custom modules is part of the EdgeAgent twin. Review the [limitation on module twin size](../iot-hub/iot-hub-devguide-module-twins.md#module-twin-size).

If you deploy a large number of modules, you might exhaust this twin size limit. Consider some common mitigations to this hard limit:

- Store any configuration in the custom module twin, which has its own limit.
- Store some configuration that points to a non-space-limited location (that is, to a blob store).

## Container management

* **Important**
  * Use tags to manage versions
* **Helpful**
  * Store runtime containers in your private registry

### Use tags to manage versions

A tag is a docker concept that you can use to distinguish between versions of docker containers. Tags are suffixes like **1.1** that go on the end of a container repository. For example, **mcr.microsoft.com/azureiotedge-agent:1.1**. Tags are mutable and can be changed to point to another container at any time, so your team should agree on a convention to follow as you update your module images moving forward.

Tags also help you to enforce updates on your IoT Edge devices. When you push an updated version of a module to your container registry, increment the tag. Then, push a new deployment to your devices with the tag incremented. The container engine will recognize the incremented tag as a new version and will pull the latest module version down to your device.

#### Tags for the IoT Edge runtime

The IoT Edge agent and IoT Edge hub images are tagged with the IoT Edge version that they are associated with. There are two different ways to use tags with the runtime images:

* **Rolling tags** - Use only the first two values of the version number to get the latest image that matches those digits. For example, 1.1 is updated whenever there's a new release to point to the latest 1.1.x version. If the container runtime on your IoT Edge device pulls the image again, the runtime modules are updated to the latest version. Deployments from the Azure portal default to rolling tags. *This approach is suggested for development purposes.*

* **Specific tags** - Use all three values of the version number to explicitly set the image version. For example, 1.1.0 won't change after its initial release. You can declare a new version number in the deployment manifest when you're ready to update. *This approach is suggested for production purposes.*

### Store runtime containers in your private registry

You know about storing your container images for custom code modules in your private Azure registry, but you can also use it to store public container images such as for the edgeAgent and edgHub runtime modules. Doing so may be required if you have very tight firewall restrictions as these runtime containers are stored in the Microsoft Container Registry (MCR).

Obtain the images with the Docker pull command to place in your private registry. You will need to specify the container version during the pull operation, find the latest container version at container description page as below, and replace the version in the pull command if needed. Be aware that you will need to update the images with each new release of IoT Edge runtime.

| IoT Edge runtime container | Docker pull command |
| --- | --- |
| [Azure IoT Edge Agent](https://hub.docker.com/_/microsoft-azureiotedge-agent) | `docker pull mcr.microsoft.com/azureiotedge-agent:<VERSION_TAG>` |
| [Azure IoT Edge Hub](https://hub.docker.com/_/microsoft-azureiotedge-hub) | `docker pull mcr.microsoft.com/azureiotedge-hub:<VERSION_TAG>` |

Next, be sure to update the image references in the deployment.template.json file for the edgeAgent and edgeHub system modules. Replace `mcr.microsoft.com` with your registry name and server for both modules.

* edgeAgent:

    `"image": "<registry name and server>/azureiotedge-agent:1.1",`

* edgeHub:

    `"image": "<registry name and server>/azureiotedge-hub:1.1",`

## Networking

* **Helpful**
  * Review outbound/inbound configuration
  * Allow connections from IoT Edge devices
  * Configure communication through a proxy

### Review outbound/inbound configuration

Communication channels between Azure IoT Hub and IoT Edge are always configured to be outbound. For most IoT Edge scenarios, only three connections are necessary. The container engine needs to connect with the container registry (or registries) that holds the module images. The IoT Edge runtime needs to connect with IoT Hub to retrieve device configuration information, and to send messages and telemetry. And if you use automatic provisioning, IoT Edge needs to connect to the Device Provisioning Service. For more information, see [Firewall and port configuration rules](troubleshoot.md#check-your-firewall-and-port-configuration-rules).

### Allow connections from IoT Edge devices

If your networking setup requires that you explicitly permit connections made from IoT Edge devices, review the following list of IoT Edge components:

* **IoT Edge agent** opens a persistent AMQP/MQTT connection to IoT Hub, possibly over WebSockets.
* **IoT Edge hub** opens a single persistent AMQP connection or multiple MQTT connections to IoT Hub, possibly over WebSockets.
* **IoT Edge service** makes intermittent HTTPS calls to IoT Hub.

In all three cases, the fully-qualified domain name (FQDN) would match the pattern `\*.azure-devices.net`.

Additionally, the **Container engine** makes calls to container registries over HTTPS. To retrieve the IoT Edge runtime container images, the FQDN is `mcr.microsoft.com`. The container engine connects to other registries as configured in the deployment.

This checklist is a starting point for firewall rules:

   | FQDN (\* = wildcard) | Outbound TCP Ports | Usage |
   | ----- | ----- | ----- |
   | `mcr.microsoft.com`  | 443 | Microsoft Container Registry |
   | `global.azure-devices-provisioning.net`  | 443 | [Device Provisioning Service](../iot-dps/about-iot-dps.md) access (optional) |
   | `\*.azurecr.io` | 443 | Personal and third-party container registries |
   | `\*.blob.core.windows.net` | 443 | Download Azure Container Registry image deltas from blob storage |
   | `\*.azure-devices.net` | 5671, 8883, 443<sup>1</sup> | IoT Hub access |
   | `\*.docker.io`  | 443 | Docker Hub access (optional) |

<sup>1</sup>Open port 8883 for secure MQTT or port 5671 for secure AMQP. If you can only make connections via port 443 then either of these protocols can be run through a WebSocket tunnel.

Since the IP address of an IoT hub can change without notice, always use the FQDN to allow-list configuration. To learn more, see [Understanding the IP address of your IoT hub](../iot-hub/iot-hub-understand-ip-address.md).

Some of these firewall rules are inherited from Azure Container Registry. For more information, see [Configure rules to access an Azure container registry behind a firewall](../container-registry/container-registry-firewall-access-rules.md).

> [!NOTE]
> To provide a consistent FQDN between the REST and data endpoints, beginning **June 15, 2020** the Microsoft Container Registry data endpoint will change from `*.cdn.mscr.io` to `*.data.mcr.microsoft.com`  
> For more information, see [Microsoft Container Registry client firewall rules configuration](https://github.com/microsoft/containerregistry/blob/master/client-firewall-rules.md)

If you don't want to configure your firewall to allow access to public container registries, you can store images in your private container registry, as described in [Store runtime containers in your private registry](#store-runtime-containers-in-your-private-registry).

### Configure communication through a proxy

If your devices are going to be deployed on a network that uses a proxy server, they need to be able to communicate through the proxy to reach IoT Hub and container registries. For more information, see [Configure an IoT Edge device to communicate through a proxy server](how-to-configure-proxy-support.md).

## Solution management

* **Helpful**
  * Set up logs and diagnostics
  * Place limits on log size
  * Consider tests and CI/CD pipelines

### Set up logs and diagnostics

On Linux, the IoT Edge daemon uses journals as the default logging driver. You can use the command-line tool `journalctl` to query the daemon logs.

<!-- 1.1 -->
:::moniker range="iotedge-2018-06"
On Windows, the IoT Edge daemon uses PowerShell diagnostics. Use `Get-IoTEdgeLog` to query logs from the daemon. IoT Edge modules use the JSON driver for logging, which is the  default.  

```powershell
. {Invoke-WebRequest -useb aka.ms/iotedge-win} | Invoke-Expression; Get-IoTEdgeLog
```

:::moniker-end
<!-- end 1.1 -->

<!--1.2-->
:::moniker range=">=iotedge-2020-11"

Starting with version 1.2, IoT Edge relies on multiple daemons. While each daemon's logs can be individually queried with `journalctl`, the `iotedge system` commands provide a convenient way to query the combined logs.

* Consolidated `iotedge` command:

  ```bash
  sudo iotedge system logs
  ```

* Equivalent `journalctl` command:

  ```bash
  journalctl -u aziot-edge -u aziot-identityd -u aziot-keyd -u aziot-certd -u aziot-tpmd
  ```

:::moniker-end

When you're testing an IoT Edge deployment, you can usually access your devices to retrieve logs and troubleshoot. In a deployment scenario, you may not have that option. Consider how you're going to gather information about your devices in production. One option is to use a logging module that collects information from the other modules and sends it to the cloud. One example of a logging module is [logspout-loganalytics](https://github.com/veyalla/logspout-loganalytics), or you can design your own.

### Place limits on log size

By default the Moby container engine does not set container log size limits. Over time this can lead to the device filling up with logs and running out of disk space. Consider the following options to prevent this:

#### Option: Set global limits that apply to all container modules

You can limit the size of all container logfiles in the container engine log options. The following example sets the log driver to `json-file` (recommended) with limits on size and number of files:

```JSON
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    }
}
```

Add (or append) this information to a file named `daemon.json` and place it in the following location:

<!-- 1.1 -->
:::moniker range="iotedge-2018-06"
| Platform | Location |
| -------- | -------- |
| Linux | `/etc/docker/` |
| Windows | `C:\ProgramData\iotedge-moby\config\` |
:::moniker-end
<!-- end 1.1 -->

<!-- 1.2 -->
:::moniker range=">=iotedge-2020-11"

* `/etc/docker/`

:::moniker-end
<!-- end 1.2 -->

The container engine must be restarted for the changes to take effect.

#### Option: Adjust log settings for each container module

You can do so in the **createOptions** of each module. For example:

```yml
"createOptions": {
    "HostConfig": {
        "LogConfig": {
            "Type": "json-file",
            "Config": {
                "max-size": "10m",
                "max-file": "3"
            }
        }
    }
}
```

#### Additional options on Linux systems

* Configure the container engine to send logs to `systemd` [journal](https://docs.docker.com/config/containers/logging/journald/) by setting `journald` as the default logging driver.

* Periodically remove old logs from your device by installing a logrotate tool. Use the following file specification:

   ```txt
   /var/lib/docker/containers/*/*-json.log{
       copytruncate
       daily
       rotate7
       delaycompress
       compress
       notifempty
       missingok
   }
   ```

### Consider tests and CI/CD pipelines

For the most efficient IoT Edge deployment scenario, consider integrating your production deployment into your testing and CI/CD pipelines. Azure IoT Edge supports multiple CI/CD platforms, including Azure DevOps. For more information, see [Continuous integration and continuous deployment to Azure IoT Edge](how-to-continuous-integration-continuous-deployment.md).

## Security considerations

* **Important**
  * Manage access to your container registry
  * Limit container access to host resources

### Manage access to your container registry

Before you deploy modules to production IoT Edge devices, ensure that you control access to your container registry so that outsiders can't access or make changes to your container images. Use a private container registry to manage container images.

In the tutorials and other documentation, we instruct you to use the same container registry credentials on your IoT Edge device as you use on your development machine. These instructions are only intended to help you set up testing and development environments more easily, and should not be followed in a production scenario.

For a more secured access to your registry, you have a choice of [authentication options](../container-registry/container-registry-authentication.md). A popular and recommended authentication is to use an Active Directory service principal that's well suited for applications or services to pull container images in an automated or otherwise unattended (headless) manner, as IoT Edge devices do. Another option is to use repository-scoped tokens, which allow you to create long or short-live identities that exist only in the Azure Container Registry they were created in and scope access to the repository level.

To create a service principal, run the two scripts as described in [create a service principal](../container-registry/container-registry-auth-service-principal.md#create-a-service-principal). These scripts do the following tasks:

* The first script creates the service principal. It outputs the Service principal ID and the Service principal password. Store these values securely in your records.

* The second script creates role assignments to grant to the service principal, which can be run subsequently if needed. We recommend applying the **acrPull** user role for the `role` parameter. For a list of roles, see [Azure Container Registry roles and permissions](../container-registry/container-registry-roles.md).

To authenticate using a service principal, provide the service principal ID and password that you obtained from the first script. Specify these credentials in the deployment manifest.

* For the username or client ID, specify the service principal ID.

* For the password or client secret, specify the service principal password.

<br>

To create repository-scoped tokens, please follow [create a repository-scoped token](../container-registry/container-registry-repository-scoped-permissions.md).

To authenticate using repository-scoped tokens, provide the token name and password that you obtained after creating your repository-scoped token. Specify these credentials in the deployment manifest.

* For the username, specify the token's username.

* For the password, specify one of the token's passwords.

> [!NOTE]
> After implementing an enhanced security authentication, disable the **Admin user** setting so that the default username/password access is no longer available. In your container registry in the Azure portal, from the left pane menu under **Settings**, select **Access Keys**.

### Limit container access to host resources

To balance shared host resources across modules, we recommend putting limits on resource consumption per module. These limits ensure that one module can't consume too much memory or CPU usage and prevent other processes from running on the device. The IoT Edge platform does not limit resources for modules by default, since knowing how much resource a given module needs to run optimally requires testing.

Docker provides some constraints that you can use to limit resources like memory and CPU usage. For more information, see [Runtime options with memory, CPUs, and GPUs](https://docs.docker.com/config/containers/resource_constraints/).

These constraints can be applied to individual modules by using create options in deployment manifests. For more information, see [How to configure container create options for IoT Edge modules](how-to-use-create-options.md).

## Next steps

* Learn more about [IoT Edge automatic deployment](module-deployment-monitoring.md).
* See how IoT Edge supports [Continuous integration and continuous deployment](how-to-continuous-integration-continuous-deployment.md).
