# Exercise - Create and manage network security groups

As the solution architect for the manufacturing company, you now want to start moving the ERP app and database servers to Azure. As a first step, you're going to test your network security plan using two of your servers.

In this unit, you'll configure a network security group and security rules to restrict network traffic to specific servers. You want your app server to be able to connect to your database server over HTTP. You don't want the database server to be able to use HTTP to connect to the app server.

Diagram of exercise scenario network security groups.

## Create a virtual network and network security group

First, you'll create the virtual network, and subnets for your server resources. You'll then create a network security group.

1. In Azure Cloud Shell, run the following command to assign the sandbox resource group to the variable `rg`:

```shell
rg=[sandbox resource group name]
```

2. To create the ERP-servers virtual network and the Applications subnet, run the following command in Cloud Shell:

```shell
az network vnet create \
    --resource-group $rg \
    --name ERP-servers \
    --address-prefixes 10.0.0.0/16 \
    --subnet-name Applications \
    --subnet-prefixes 10.0.0.0/24
```

3. To create the Databases subnet, run the following command in Cloud Shell:

```shell
az network vnet subnet create \
    --resource-group $rg \
    --vnet-name ERP-servers \
    --address-prefixes 10.0.1.0/24 \
    --name Databases
```

4. To create the ERP-SERVERS-NSG network security group, run the following command in Cloud Shell:

```shell
az network nsg create \
    --resource-group $rg \
    --name ERP-SERVERS-NSG
```

## Create VMs running Ubuntu

Next, you'll create two VMs called AppServer and DataServer. You deploy AppServer to the Applications subnet, and DataServer to the Databases subnet. Add the VM network interfaces to the ERP-SERVERS-NSG network security group. Then, to test your network security group, use these VMs.

1. To build the AppServer VM, in Cloud Shell, run the following command. For the admin account, replace `<password>` with a complex password.

```shell
wget -N https://raw.githubusercontent.com/MicrosoftDocs/mslearn-secure-and-isolate-with-nsg-and-service-endpoints/master/cloud-init.yml && \
az vm create \
    --resource-group $rg \
    --name AppServer \
    --vnet-name ERP-servers \
    --subnet Applications \
    --nsg ERP-SERVERS-NSG \
    --image UbuntuLTS \
    --size Standard_DS1_v2 \
     --generate-ssh-keys \
    --admin-username azureuser \
    --custom-data cloud-init.yml \
    --no-wait \
    --admin-password <password>
```

2. To build the DataServer VM, in Cloud Shell, run the following command. For the admin account, replace `<password>` with a complex password.

```shell
az vm create \
    --resource-group $rg \
    --name DataServer \
    --vnet-name ERP-servers \
    --subnet Databases \
    --nsg ERP-SERVERS-NSG \
    --size Standard_DS1_v2 \
    --image UbuntuLTS \
    --generate-ssh-keys \
    --admin-username azureuser \
    --custom-data cloud-init.yml \
     --no-wait \
    --admin-password <password>
```

3. It can take several minutes for the VMs to be in a running state. To confirm that the VMs are running, run the following command in Cloud Shell:

```shell
az vm list \
    --resource-group $rg \
    --show-details \
    --query "[*].{Name:name, Provisioned:provisioningState, Power:powerState}" \
    --output table
```

When your VM creation is complete, you should see the following output.


Name        Provisioned    Power
----------  -------------  ----------
AppServer   Succeeded      VM running
DataServer  Succeeded      VM running

## Check default connectivity

Now, you'll try to open a Secure Shell (SSH) session to each of your VMs. Remember, so far you've deployed a network security group with default rules.

1. To connect to your VMs, use SSH directly from Cloud Shell. To do this, you need the public IP addresses that have been assigned to your VMs. To list the IP addresses that you'll use to connect to the VMs, run the following command in Cloud Shell:

```shell
az vm list \
    --resource-group $rg \
    --show-details \
    --query "[*].{Name:name, PrivateIP:privateIps, PublicIP:publicIps}" \
    --output table
```

2. To make it easier to connect to your VMs during the rest of this exercise, assign the public IP addresses to variables. To save the public IP address of AppServer and DataServer to a variable, run the following command in Cloud Shell:

```bash
APPSERVERIP="$(az vm list-ip-addresses \
                 --resource-group $rg \
                 --name AppServer \
                 --query "[].virtualMachine.network.publicIpAddresses[*].ipAddress" \
                 --output tsv)"

DATASERVERIP="$(az vm list-ip-addresses \
                 --resource-group $rg \
                 --name DataServer \
                 --query "[].virtualMachine.network.publicIpAddresses[*].ipAddress" \
                 --output tsv)"
```

3. To check whether you can connect to your AppServer VM, run the following command in Cloud Shell:

```shell
ssh azureuser@$APPSERVERIP -o ConnectTimeout=5
```

You'll get a `Connection timed out` message.

4. To check whether you can connect to your DataServer VM, run the following command in Cloud Shell:

```shell
ssh azureuser@$DATASERVERIP -o ConnectTimeout=5
```

You'll get the same connection failure message.

Remember that the default rules deny all inbound traffic into a virtual network, unless this traffic is coming from the same virtual network. The Deny All Inbound rule blocked the inbound SSH connections you just attempted.

### Inbound

Name	Priority	Source IP	Destination IP	Access
Allow VNet Inbound	65000	VIRTUAL_NETWORK	VIRTUAL_NETWORK	Allow
Deny All Inbound	65500	*	*	Deny

## Create a security rule for SSH

As you've now experienced, the default rules in your ERP-SERVERS-NSG network security group include a Deny All Inbound rule. You'll now add a rule so that you can use SSH to connect to AppServer and DataServer.

1. YOUR TURN - create a new inbound security rule to enable SSH access.

2. To check whether you can now connect to your AppServer VM, run the following command in Cloud Shell:

```shell
ssh azureuser@$APPSERVERIP -o ConnectTimeout=5
```

The network security group rule might take a minute or two to take effect. If you receive a connection failure message, try again.

3. You should now be able to connect. After the `Are you sure you want to continue connecting (yes/no)?` message, enter `yes`.

4. Enter the password you defined when you created the VM.

5. To close the AppServer session, enter `exit`.

6. To check whether you can now connect to your DataServer VM, run the following command in Cloud Shell:

```shell
ssh azureuser@$DATASERVERIP -o ConnectTimeout=5
```

7. You should now be able to connect. After the `Are you sure you want to continue connecting (yes/no)?` message, enter `yes`.

8. Enter the password you defined when you created the VM.

9. To close the DataServer session, enter `exit`.

## Create a security rule to prevent web access

Now, add a rule so that AppServer can communicate with DataServer over HTTP, but DataServer can't communicate with AppServer over HTTP. These are the internal IP addresses for these servers:

Server name	IP address
AppServer	10.0.0.4
DataServer	10.0.1.4

1. YOUR TURN - create a new inbound security rule to deny HTTP access over port 80.

## Test HTTP connectivity between virtual machines

Here, you'll check if your new rule works. AppServer should be able to communicate with DataServer over HTTP. DataServer shouldn't be able to communicate with AppServer over HTTP.

1. To connect to your AppServer VM, in Cloud Shell, run the following command. Check if AppServer can communicate with DataServer over HTTP.

```shell
ssh -t azureuser@$APPSERVERIP 'wget http://10.0.1.4; exit; bash'
```

2. Enter the password you defined when you created the VM.

3. The response should include a `200 OK` message.

4. To connect to your DataServer VM, in Cloud Shell, run the following command. Check if DataServer can communicate with AppServer over HTTP.

```shell
ssh -t azureuser@$DATASERVERIP 'wget http://10.0.0.4; exit; bash'
```

5. Enter the password you defined when you created the VM.

6. This shouldn't succeed, because you've blocked access over port 80. After several minutes, you should get a `Connection timed out` message. To stop the command before the timeout, press `Ctrl+C`.