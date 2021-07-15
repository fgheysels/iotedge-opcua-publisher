# Deploy IoT Edge OPC UA Publisher module to IoT Edge device (PoC)

## Abstract

This document provides a step-by-step approach for deploying the [OPC UA publisher IoT Edge module](https://github.com/Azure/iot-edge-opc-publisher) on an IoT Edge device.

## Create an IoT Edge Device

### Register a new IoT Edge device in IoT Hub.  

This can be done via the Azure Portal, or via the commandline, by executing this CLI command:

```azcli
az iot hub device-identity create --device-id <my_device_id> --edge-enabled --hub-name <iothubname>
```

### Deploy an Azure VM which will act as the IoT Edge device

For this PoC, an Azure VM will be used which hosts the IoT Edge runtime.
We'll provision the VM and make sure that we can logon to it via ssh.

#### Generate SSH keys

First, a ssh keyset must be generated which will be used to get remote access to the VM.

Execute this command:

```powershell
ssh-keygen -m PEM -t rsa -b 4096 -f opcua_iotedge_vm.pem
```

This commands generates 2 files:
- opcua_iotedge_vm.pem:  the private key that should be stored in a secured location
- opcua_iotedge_vm.pub: the public key for this keyset.

#### Deploy VM with IoT Edge pre-installed

There exists an ARM template which deploys a Linux virtual machine which has IoT Edge pre-installed.  The VM can be easily provisioned by executing this command:

```powershell
az deployment group create `
  --resource-group <resourcegroup where the VM must be deployed> `
  --template-uri "https://raw.githubusercontent.com/Azure/iotedge-vm-deploy/1.2.0/edgeDeploy.json" `
  --parameters dnsLabelPrefix='<name of the VM>' `
  --parameters adminUsername='<username>' `
  --parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id <my_device_id> --hub-name <iothubname> -o tsv) `
  --parameters authenticationType='sshPublicKey' `
  --parameters adminPasswordOrKey="<paste contents of ssh.pub file>"
```

Once the resources have been deployed, you'll see the 'outputs' being written to the screen.  Take note of the publicFQND and/or publicSSH properties.  (Note, this information can also be found again via the Azure Portal)

#### SSH to the VM

Once the virtual machine has been created, you can log on to it via ssh by executing this command:

```powershell
ssh -i <myprivatekeyfile> <username>@<fqdn of VM>
```

for instance:

```powershell
ssh -i opcua_iotedge_vm.pem fgheysels@myvirtualmachine.westeurope.cloudapp.azure.com
```

#### Check if IoT Edge is running on the VM

Once logged in to the VM, check if IoT Edge is running by executing these commands:

```bash
sudo iotedge check
```

```bash
iotedge list
```

### Deploy OPC UA publisher module to the IoT Edge device

[This documentation page](https://github.com/Azure/iot-edge-opc-publisher) on GitHub describes how the OPC UA publisher module can be deployed on an IoT Edge device.

We can also use the command line and a deployment manifest to do the same.

#### Prepare the Edge Device

If there are already multiple IoT Edge devices registered in the IoT Hub, it is advised to first define a tag in the Device Twin of the Edge Device in IoT Hub.  We do not want to publish the deployment manifest to all devices in IoT Hub, instead, we only want to target one specific device in this PoC.

- Navigate to the IoT Edge blade of your IoT Hub in the Azure Portal
- Select the IoT Edge device that is used in this PoC which will host the OPC UA publisher
- Open the Device Twin
- Add a property `tags` to the JSON document, and specify a specific tag where we can filter on.
  
  ```json
  "version": 3,
    "tags": {
        "opcua": true
    },
    "properties": {
  ```

#### Deploy the IoT Edge manifest to IoT Hub

This repository contains a file `deployment.json`.  This file is a deployment manifest which describes which modules must be deployed.  Next to the edgeHub and edgeAgent system modules, this manifest also describes that the OPC UA Publisher module must be deployed.

Execute this command to deploy the modules to the edge device:

```
$deploymentName = "opcua_$(Get-Date -Format yyyyMMdd)"

az iot edge deployment create -d $deploymentName --content .\deployment.json -l "<iothubconnectionstring>" --target-condition "tags.opcua = true"
```

Note that the target-condition works on the `tags` property.  This makes sure that only devices that have this specific tag are targetted by the deployment.

After a few moments, you'd see that the deployment targets the device that has been registered and the deployment should be applied to that device.

When executing this command on the VM that runs IoT Edge:

```
iotedge list
```

You should see that IoT edge is running:
 - edgeHub 1.2
 - edgeAgent 1.2
 - OPCPublisher


Now, the IoT Edge device is able to listen to OPC UA events.

## Setup a Mock which simulates an OPC UA server which simulates OPC UA data being sent

There exists a GitHub repository which contains a OPC UA Server simulator.  It can be found [here](https://github.com/Azure-Samples/iot-edge-opc-plc).
Deploying it is just a matter of clicking the 'Deploy to Azure' button.

## Connect the OPC UA Publisher module to the OPC UA Server Simulator

The OPCPublisher module is currently not receiving any data since it is not configured yet.
Moreover, if the command `iotedge logs OPCPublisher` is executed, you can see that the module is not functioning correctly:  there is no configuration file available.

The containerCreate options of the OPCPublisher module state:

```json
{
  "HostConfig": {
    "Binds": [
      "/iiotedge:/appdata"
    ]
  },
  "Hostname": "opcpublisher",
  "Cmd": [
    "--pf=./publishednodes.json",
    "--aa"
  ]
}
```

The configuration file must be named `publishednodes.json` and must exist in the `/appdata` folder that exists inside the container.
The `/iiotedge` folder which should exist on the Host (The VM that runs iotedge) is mounted to the `/appdata` folder in the container.

### Configuration

The `publishednodes.json` should look like this:

```json
[
  {
    "EndpointUrl": "opc.tcp://aci-bekaert-g4mwza6-plc1.westeurope.azurecontainer.io:50000",
    "UseSecurity": false,
    "OpcNodes": [
      { "Id": "ns=2;s=AlternatingBoolean" },
      { "Id": "ns=2;s=DipData" },
      { "Id": "ns=2;s=NegativeTrendData" },
      { "Id": "ns=2;s=PositiveTrendData" },
      { "Id": "ns=2;s=RandomSignedInt32" },
      { "Id": "ns=2;s=RandomUnsignedInt32" },
      { "Id": "ns=2;s=SpikeData" },
      { "Id": "ns=2;s=StepUp" }
    ]
  }
]
```

The `EndpointUrl` can be found in the logs of the OPC UA Server Simulator that is running in the Azure Container Instance that has been deployed.
Actually, the complete JSON of how this file should look like, can be found in the logs of the OPC UA Server Simulator.