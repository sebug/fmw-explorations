# Fusion Middleware Explorations
Setting up two Oracle databases and two Fusion Middleware machines to test
out some stuff.

Tutorial followed: https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/oracle/oracle-database-quick-create

	az group create --name fmw-explorations --location westeurope

## Database Machines
We want two database machines to experiment with a completely separate
environment (they can see each other, that's fine, we just want them
to be environmentally (IP, name) different).

	az vm create \
    --resource-group fmw-explorations \
    --name vmoracle19c1 \
    --image Oracle:oracle-database-19-3:oracle-database-19-0904:latest \
    --size Standard_DS2_v2 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --public-ip-address-allocation static \
    --public-ip-address-dns-name vmoracle19c1

	az vm disk attach --name oradata01 --new --resource-group fmw-explorations --size-gb 64 --sku StandardSSD_LRS --vm-name vmoracle19c1

	az network nsg rule create \
    --resource-group fmw-explorations \
    --nsg-name vmoracle19c1NSG \
    --name allow-oracle \
    --protocol tcp \
    --priority 1001 \
    --destination-port-range 1521

	az network nsg rule create \
    --resource-group fmw-explorations \
    --nsg-name vmoracle19c1NSG \
    --name allow-em \
    --protocol tcp \
    --priority 1002 \
    --destination-port-range 5502

Adding a second VM

	az vm create \
    --resource-group fmw-explorations \
    --name vmoracle19c2 \
    --image Oracle:oracle-database-19-3:oracle-database-19-0904:latest \
    --size Standard_DS2_v2 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --public-ip-address-allocation static \
    --public-ip-address-dns-name vmoracle19c2

	az vm disk attach --name oradata02 --new --resource-group fmw-explorations --size-gb 64 --sku StandardSSD_LRS --vm-name vmoracle19c2

	az network nsg rule create \
    --resource-group fmw-explorations \
    --nsg-name vmoracle19c2NSG \
    --name allow-oracle \
    --protocol tcp \
    --priority 1001 \
    --destination-port-range 1521

	az network nsg rule create \
    --resource-group fmw-explorations \
    --nsg-name vmoracle19c2NSG \
    --name allow-em \
    --protocol tcp \
    --priority 1002 \
    --destination-port-range 5502

### Initial setup of the VMs
On both VMs, connect using

	ssh azureuser@public-ip-address

And execute the following commands to finish setup:

	sudo su -
	ls -alt /dev/sd*|head -1

This gives us the name of the disk to format (in the example below sdc)

	parted /dev/sdc mklabel gpt
	parted -a optimal /dev/sdc mkpart primary 0GB 64GB
	parted /dev/sdc print

Ensure that the end and size are 64 GB, number 1.

	mkfs -t ext4 /dev/sdc1
	mkdir /u02
	mount /dev/sdc1 /u02
	chmod 777 /u02
	echo "/dev/sdc1               /u02                    ext4    defaults        0 0" >> /etc/fstab

Creates a file system, mounts it and gives access to everyone.

Now we'll have to modify /etc/hosts

	echo "public-ip-address vmoracle19c1.westeurope.cloudapp.azure.com vmoracle19c1" >> /etc/hosts

replacing vmoracle19c1 with vmoracle19c2 the second time around

	sed -i 's/$/\.westeurope\.cloudapp\.azure\.com &/' /etc/hostname

Open firewall:

	firewall-cmd --zone=public --add-port=1521/tcp --permanent
	firewall-cmd --zone=public --add-port=5502/tcp --permanent
	firewall-cmd --reload

## Create the databases
On both machines, ssh in again, then

	sudo su - oracle
	lsnrctl start # May fail, come back to later
	mkdir /u02/oradata

Now create the database, replace 1 with 2 for the second machine

	dbca -silent -createDatabase -templateName General_Purpose.dbc -gdbname oratest1 -sid oratest1 -responseFile NO_VALUE -characterSet AL32UTF8 -sysPassword OraPasswd1 -systemPassword OraPasswd1 -createAsContainerDatabase false -databaseType MULTIPURPOSE -automaticMemoryManagement false -storageType FS -datafileDestination "/u02/oradata/" -ignorePreReqs
	export ORACLE_SID=oratest1
	echo "export ORACLE_SID=oratest1" >> ~oracle/.bashrc

