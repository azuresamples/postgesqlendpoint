## はじめに  
既存 VNet にサブネットを追加し、そこにサービスエンドポイントを作って、VM からデータベースに接続してみます。  
従来のファイアウォールルールどどう違うのか、ポータルからの見え方を確認してみます。  

## 作ってみる  
Azure CLI でやってみます。  
* Azure Database for PostgreSQL を作成
* サブネットにサービスエンドポイントを付与
* Azure Database for PostgreSQL に  Connection security（VNET rules）を作成 

以上のような手順で作成します。  
作成に使った Azure CLI は以下のとおりです。 

## Azure Database for PostgreSQL を作る  
Azure Database for PostgreSQL を作成します。   
```
az postgres server create --resource-group gasu-rsg \
                          --name gasupgserver-20191231 \
                          --location japaneast \
                          --admin-user sugaadmin \
                          --admin-password GASU#E5p9dZbH \
                          --sku-name GP_Gen5_2 \
                          --ssl-enforcement Enabled \
                          --storage-size 5120 \
                          --backup-retention 7 \
                          --version 10

az postgres db create --name testdb \
                      --resource-group gasu-rsg \
                      --server-name gasupgserver-20191231 \
                      --charset UTF8 \
                      --collation C
```
データベース`testdb`がちゃんとできているかどうか確認します。  
```
$ az postgres db list --resource-group gasu-rsg                     --server-name gasupgserver-20191231 -o table
Charset    Collation                   Name               ResourceGroup
---------  --------------------------  -----------------  ---------------
UTF8       English_United States.1252  postgres           gasu-rsg
UTF8       English_United States.1252  azure_maintenance  gasu-rsg
UTF8       English_United States.1252  azure_sys          gasu-rsg
UTF8       C                           testdb             gasu-rsg
```
Collation C のデータベースが追加されています。  

## サービスエンドポイントを作る  
次に既存サブネットにサービスエンドポイントを付与します。このサブネットは空です。    
まず、当該リージョンで利用できるサービスエンドポイントの種類を確認します。  
```
# Get available service endpoints for Azure region output is JSON
# Use the command below to get the list of services supported for endpoints, for an Azure region, say "japaneast".
az network vnet list-endpoint-services \
-l japaneast \
-o table
```  

```
Name
------------------------------
Microsoft.Storage
Microsoft.Sql
Microsoft.AzureActiveDirectory
Microsoft.AzureCosmosDB
Microsoft.Web
Microsoft.KeyVault
Microsoft.EventHub
Microsoft.ServiceBus
Microsoft.ContainerRegistry
```  
`Microsoft.Sql`のサービスエンドポイントを作ります。サブネットが既存なので、updateコマンドを使います。  
```
# Creates the service endpoint
az network vnet subnet update \
-g gasu-rsg \
-n gasu-subnet6 \
--vnet-name gasu-vnet \
--service-endpoints Microsoft.SQL
```  


サブネットに`Microsoft.SQL`のサービスエンドポイントが構成されているかどうか確認します。 
```
# View service endpoints configured on a subnet
az network vnet subnet show \
-g gasu-rsg \
-n gasu-subnet6 \
--vnet-name gasu-vnet
```  
 
```
$ az network vnet subnet show \
> -g gasu-rsg \
> -n gasu-subnet6 \
> --vnet-name gasu-vnet
{
  "addressPrefix": "10.3.7.0/24",
  "addressPrefixes": null,
  "delegations": [],
  "etag": "W/\"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx\"",
  "id": "/subscriptions/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy/resourceGroups/gasu-rsg/providers/Microsoft.Network/virtualNetworks/gasu-vnet/subnets/gasu-subnet6",
  "ipConfigurationProfiles": null,
  "ipConfigurations": [
    {
      "etag": null,
      "id": "/subscriptions/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy/resourceGroups/gasu-rsg/providers/Microsoft.Network/networkInterfaces/centosservicendpoint677/ipConfigurations/ipconfig1",
      "name": null,
      "privateIpAddress": null,
      "privateIpAllocationMethod": null,
      "provisioningState": null,
      "publicIpAddress": null,
      "resourceGroup": "gasu-rsg",
      "subnet": null
    }
  ],
  "name": "gasu-subnet6",
  "natGateway": null,
  "networkSecurityGroup": null,
  "privateEndpointNetworkPolicies": "Enabled",
  "privateEndpoints": null,
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "purpose": null,
  "resourceGroup": "gasu-rsg",
  "resourceNavigationLinks": null,
  "routeTable": null,
  "serviceAssociationLinks": null,
  "serviceEndpointPolicies": null,
  "serviceEndpoints": [
    {
      "locations": [
        "japaneast"
      ],
      "provisioningState": "Succeeded",
      "service": "Microsoft.Sql"
    }
  ],
  "type": "Microsoft.Network/virtualNetworks/subnets"
}
```
## VNET rules を作る  
Connection security（VNET rules）を作成します。  
```
# Create a VNet rule on the sever to secure it to the subnet. 
# Note: resource group (-g) parameter is where the database exists. 
# VNet resource group if different should be specified using subnet id (URI) instead of subnet, VNet pair.
az postgres server vnet-rule create \
-n gasu-pgsqlRule \
-g gasu-rsg \
-s gasupgserver-20191231 \
--vnet-name gasu-vnet \
--subnet gasu-subnet6
```
ポータルから Connection Security を確認してみます。  
![2019-09-02_12h54_45](uploads/390d86888add2426d7dd8009e84d9de4/2019-09-02_12h54_45.png)  

ファイアウォールルールではないセキュリティルール、VNET rule ができています。  
VNet 統合のルールです。  
それでは、gasu-subnet6 に作った VM （10.3.7.4）から gasupgserver-20191231 に接続してみます。  
ファイアウォールルールはありませんので、許可するグローバル IP アドレスを書きません。  
VNet ルールでは、接続対象のサブネット（10.3.7.0/24）を書きます。サブネット内の VM はパブリック IP アドレス無しで、Azure Database for PostgreSQL に接続することができます。    
![2019-09-02_11h44_03](uploads/e58281c559f903485adb1cb8ade7ed91/2019-09-02_11h44_03.png)    

正しく接続できています。  

## おわりに  
以上のように VNet 統合ではグローバル IP アドレスを接続ルールに記述しない設定です。ファイアーウォールルールでは、接続する VM 側がグローバル IP アドレスを持っていなければならないといった制約があり、気分的には VNet 統合の方が安心できます。（VM に グローバル IP アドレスを付与せずに L4 ロードバランサを間に挟むといった裏技は存在します）  
