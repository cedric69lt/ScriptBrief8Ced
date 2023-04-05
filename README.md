# ScriptBrief8Ced

#! /bin/bash


# Load balance VMs across availability zones


# Variable block


let "randomIdentifier=$RANDOM*$RANDOM"


location=francecentral
vNet="msdocs-vnet-lb-myvnet$randomIdentifier"
subnet="msdocs-subnet-lb-subnet$randomIdentifier"
loadBalancerPublicIp="msdocs-public-ip-lb-$randomIdentifier"
ipSku=Standard
zone="1 2 3"
loadBalancer="msdocs-load-balancer-loadbalancer$randomIdentifier"
frontEndIp="msdocs-front-end-ip-lb-$randomIdentifier"
backEndPool="msdocs-back-end-pool-lb-$randomIdentifier"
probe80="msdocs-port80-health-probe-lb-$randomIdentifier"
loadBalancerRuleWeb="msdocs-load-balancer-rule-port80-$randomIdentifier"
loadBalancerRuleSSH="msdocs-load-balancer-rule-port22-$randomIdentifier"
networkSecurityGroup="msdocs-network-security-group-lb-networksecuritygroup"
networkSecurityGroupRuleSSH="msdocs-network-security-rule-port22-lb-$randomIdentifier"
networkSecurityGroupRuleWeb="msdocs-network-security-rule-port80-lb-$randomIdentifier"
nic="msdocs-nic-lb-$randomIdentifier"
image=UbuntuLTS
mdbserv=mdbserv$randomIdentifier


myNATgateway=msdocs-NAT-gateway$randomIdentifier
myNATgatewayIP=msdocs-NAT-gateway-IP$randomIdentifier




read -p "Entrez le nom de votre Groupe de Ressource: " GroupeDeRessource


read -p "Entrez le nom de votre VM: " vm


read -p "Entrez votre nom d'Utilisateur: " azureuser


read -p "Entrez votre nom d'utilsateur pour MariaDB: " UserMDB


# read -p "Entrez votre mot de passe pour MariaDB: "




# Creer un groupe de ressource
echo "Creating $GroupeDeRessource in France Central"


az group create \
    --name $GroupeDeRessource \
    --location $location




# Créer un vnet et un sous réseau.
echo "Creating $vNet and $subnet"


az network vnet create \
    --resource-group $GroupeDeRessource \
    --name $vNet \
    --location $location \
    --address-prefixes 10.1.0.0/16 \
    --subnet-name $subnet
    --subnet-prefixes 10.1.0.0/24


# Créer un zonal Standard avec une adresse ip publique pour le load balancer.
echo "Creating $loadBalancerPublicIp"


az network public-ip create \
    --resource-group $GroupeDeRessource \
    --name $loadBalancerPublicIp \
    --sku $ipSku \
    --zone $zone


# Créer un Azure Load Balancer.
echo "Creating $loadBalancer with $frontEndIP and $backEndPool"


az network lb create \
    --resource-group $GroupeDeRessource \
    --name $loadBalancer \
    --public-ip-address $loadBalancerPublicIp \
    --frontend-ip-name $frontEndIp \
    --backend-pool-name $backEndPool \
    --sku $ipSku


# Créer un lb probe sur le port 80.
echo "Creating $probe80 in $loadBalancer"


az network lb probe create \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer \
    --name $probe80 \
    --protocol tcp \
    --port 80


# Create une règle LB pour le port 80.
echo "Creating $loadBalancerRuleWeb for $loadBalancer"


az network lb rule create \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer \
    --name $loadBalancerRuleWeb \
    --protocol tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name $frontEndIp \
    --backend-pool-name $backEndPool \
    --probe-name $probe80
#    --disable-outbound-snat true \
#    --idle-timeout 15 \
#    --enable-tcp-reset true


# Créer 2 règles NAT pour le port 22.
echo "Creating two NAT rules named $loadBalancerRuleSSH"


  for i in `seq 1 2`
  do
    az network lb inbound-nat-rule create \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer \
    --name $loadBalancerRuleSSH$i \
    --protocol tcp \
    --frontend-port 422$i \
    --backend-port 22 \
    --frontend-ip-name $frontEndIp
  done


# Créer un groupe de sécurité de network
echo "Creating $networkSecurityGroup"


az network nsg create \
    --resource-group $GroupeDeRessource \
    --name $networkSecurityGroup


# Créer une règle pour le groupe port 22.
echo "Creating $networkSecurityGroupRuleSSH in $networkSecurityGroup for port 22"


az network nsg rule create \
    --resource-group $GroupeDeRessource \
    --nsg-name $networkSecurityGroup \
    --name $networkSecurityGroupRuleSSH \
    --protocol tcp \
    --direction inbound \
    --source-address-prefix '*' \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 22 \
    --access allow \
    --priority 1000




# On répète le processus pour le port 80.
echo "Creating $networkSecurityGroupRuleWeb in $networkSecurityGroup for port 22"


az network nsg rule create \
    --resource-group $GroupeDeRessource \
    --nsg-name $networkSecurityGroup \
    --name $networkSecurityGroupRuleWeb \
    --protocol tcp \
    --direction inbound \
    --priority 1001 \
    --source-address-prefix '*' \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 80 \
    --access allow \
    --priority 2000


# Créer 2 NICS et ont les associer avec les adresses IP publiques et NSG.
echo "Creating two NICs named $nic for $vNet and $subnet"


  for i in `seq 1 2`
  do
    az network nic create \
    --resource-group $GroupeDeRessource \
    --name $nic$i \
    --vnet-name $vNet \
    --subnet $subnet \
    --network-security-group $networkSecurityGroup \
    --lb-name $loadBalancer \
    --lb-address-pools $backEndPool \
    --lb-inbound-nat-rules $loadBalancerRuleSSH$i
  done


# Créer 2 VM avec leurs clées SSH.
echo "Creating two VMs named $vm with $nic using $image"




# read -p "Entrez le nombre de VM souhaité: " END
# for i in  $(seq 1 $END)


  for i in `seq 1 2`
  do
    az vm create \
    --resource-group $GroupeDeRessource \
    --name $vm$i \
    --zone $i \
    --nics $nic$i \
    --image $image \
    --admin-username $azureuser \
    --generate-ssh-keys \
    --no-wait
  done


  for i in `seq 1 2`
  do
    az vm open-port \
    --port 80 \
    --resource-group $GroupeDeRessource \
    --name $vm$i
  done




# Liste les VM
az vm list \
    --resource-group $GroupeDeRessource


# On teste le VM




IpPublic=$(az network public-ip show \
    --resource-group $GroupeDeRessource \
    --name $loadBalancerPublicIp \
    --query ipAddress \
    --output tsv)


echo $IpPublic


# Créer une NAT Gateway


az network public-ip create \
    --resource-group $GroupeDeRessource \
    --name $myNATgatewayIP \
    --sku Standard \
    --zone $zone


az network nat gateway create \
    --resource-group $GroupeDeRessource \
    --name $myNATgateway \
    --public-ip-addresses $myNATgatewayIP \
    --idle-timeout 10


az network vnet subnet update \
    --resource-group $GroupeDeRessource \
    --vnet-name $vNet \
    --name $subnet \
    --nat-gateway $myNATgateway


# Créer une base de données de Maria DB server


az mariadb server create \
    --resource-group $GroupeDeRessource \
    --name $mdbserv \
    --location francecentral \
    --admin-user $UserMDB \
    --admin-password PIcciNO69200!MaRaTEa? \
    --sku-name GP_Gen5_2 \
    --version 10.2


az mariadb server firewall-rule create \
    --resource-group $GroupeDeRessource \
    --server $mdbserv \
    --name AllowMyIP \
    --start-ip-address 20.74.91.62 \
    --end-ip-address 20.74.91.62


az mariadb server show \
    --resource-group $GroupeDeRessource \
    --name $mdbserv




ssh -i .ssh/id_rsa $azureuser@$IpPublic -p 4221


sudo apt update


sudo apt upgrade


sudo apt install apache2


sudo apt install mariadb-client-10.1


sudo apt install php php-mysql


CREATE DATABASE wordpress" > dbm.sql
CREATE USER wpceduser@20.74.102.126 IDENTIFIED BY 'motdepasse';
GRANT ALL ON wordpress_db.* TO wpceduser@20.74.102.126 IDENTIFIED BY 'motdepasse';


wget https://fr.wordpress.org/latest-fr_FR.tar.gz
tar xvf latest-fr_FR.tar.gz


mysql -u testvm1@mdbserv40171838 -p -h mdbserv40171838.mariadb.database.azure.com < test.sql



sudo sed -i "s/database_name_here/wordpress/" wp-config-sample.php
sudo sed -i "s/username_here/wpceduser/" wp-config-sample.php
sudo sed -i "s/password_here/motdepasse/" wp-config-sample.php
sudo sed -i "s/localhost/20.74.102.126/" wp-config-sample.php
