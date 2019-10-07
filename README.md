
#<font color="orange"> **Setup Sapience Azure Account via Terraform** </font>

### Requirements
----
1. Azure CLI locally installed.  Version 2.0.60+.  
	```
	az --version
	az extension add --name storage-preview
	```
2. Terraform locally installed
3. ansible-vault locally installed
4. Kubectl locally installed
5. Helm locally installed

### Create Storage Account and Access Key (Only do this when creating a new realm)
----
1. Create a Storage Account (manually via the Azure Portal) for Terraform remote state storage for the realm (i.e. "sapiencetfstatelab") in the "Production" Subscription / "devops" Resource Group
2. Create a blob container for the <cloud>-<region>-<environment> (i.e. "azure-us-dev")
3. Retrieve the "Access Key" for the Terraform remote state Storage Account via the Azure Portal... this will be used in Terraform "backend" blocks in each Terraform main.tf

	**SECRET** :a:

4. Create an Azure Service Principal via the Azure CLI: see Microsoft AKS documentation, Microsoft Azure CLI documentation, and Terraform documentation
	```
	az account set --subscription="<subscription_id>"
	az ad sp create-for-rbac --skip-assignment --name Terraform<Subscription>
	```
	If the sp has already been created, use `az ad sp show --id http://Terraform<Subscription>`
5. Copy and store the output of the command above

	**SECRET** :b:

6. Create realm (i.e. <realm>-<region>) in subscription through portal

7. Give Contributor role to Terraform<Subscription> service principal to the <realm>-<region> resource group created above

   ```az role assignment create --assignee <sp object id from show command above> --role Contributor --scope /subscriptions/<subscription id>/resourceGroups/<resource group i.e. lab-us>```

8. Give Storage Blob Data Contributor role to Terraform<Subscription> service principal to the "Production" subscription (so it can read/write to "sapiencetfstatelab")

   ```az role assignment create --assignee <sp object id from show command above> --role "Storage Blob Data Contributor" --scope /subscriptions/<Production subscription id>/resourceGroups/devops/providers/Microsoft.Storage/storageAccounts/<i.e. sapiencetfstatelab>```

### Create Realm and Environment Infrastructure
----
##### 1. Create Realm Infrastructure
1. Copy components from existing realm
2. Cleanup temporary directories

	```find . -type d -name ".terraform" -exec rm -rf {} +```
		
	```find . -type d -name ".local" -exec rm -rf {} +```

3. Create "storage-account"
4. Create "network"
5. Create "kubernetes"
6. Create "aks-egress"
7. Create "helm"
8. Create "kured"
9. Create "certificates"
10. Create "ingress-controller"
11. Create "monitoring"
12. Create "logging"

##### 2. Create "Dev" Environment Infrastructure
1. network
2. kubernetes-namespace
3. cronjob
4. service-bus

5. data-lake
6. database
7. canopy
8. ambassador
9. databricks
10. sapience-app-api
11. functions
12. vm
13. dns

#### 
1. Setup Kubernetes namespace
    1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
	2. Edit "terraform/environments/dev/kubernetes-namespace/main.tf"
		- Change 'key' in terraform{} block: "sapience.environment.<font color="red">dev</font>.kubernetes-namespace.terraform.tfstate"
	3. Terraform Initialize and Apply
		```
		cd terraform/environments/dev/kubernetes-namespace
		terraform init -backend-config="../../../config/backend.config"
		terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
		```
	- The new Namespace will get a default-token-* secret that needs to be added to environment.dev.tfvars. Set kubernetes_namespace_default_token.Retrieve this from the Kubernetes Portal -> Secrets.

2. Setup service-bus
    1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
	2. Edit "terraform/environments/dev/service-bus/main.tf"
		- Change 'key' in terraform{} block: "sapience.environment.<font color="red">dev</font>.service-bus.terraform.tfstate"
	3. Terraform Initialize and Apply
		```
		cd terraform/environments/dev/service-bus
		terraform init -backend-config="../../../config/backend.config"
		terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
		```

3. Setup Data Lake
    1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
	2. Edit "terraform/environments/dev/data-lake/main.tf"
		- Change 'key' in terraform{} block: "sapience.environment.<font color="red">dev</font>.data-lake.terraform.tfstate"
		- Review the "azurerm_data_lake_store_firewall_rule"(s)
	3. Terraform Initialize and Apply
		```
		cd terraform/environments/dev/data-lake
		terraform init -backend-config="../../../config/backend.config"
		terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
		```

4. Setup Databases
    1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
	2. Edit "terraform/environments/dev/database/main.tf"
		- Change 'key' in terraform{} block: "sapience.environment.<font color="red">dev</font>.database.terraform.tfstate"
	3. Terraform Initialize and Apply
		```
		cd terraform/environments/dev/database
		terraform init -backend-config="../../../config/backend.config"
		terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
		```
	4. Configure user(s) in Gremlin
	    1. Create graph database in Cosmos
			1. In Azure Portal, go to Azure Cosmos DB
			2. Click on sapience-canopy-hierarchy-${realm}-${environment}
			3. Click 'Add Graph'
			4. Set "Database id = canopy" AND "Graph Id = hierarchy"
			5. For Dev, we set Storage Capacity = 'Fixed (10GB) and Throughput = 400; Partition Key = "/name"
	    2. Execute this Gremlin query via the Cosmos portal:
			- `g.addV(label, 'User', 'ref_id', 'steve.ardis@banyanhills.com', 'name', 'steve.ardis@banyanhills.com').addE("BELONGS_TO").to(g.addV(label, 'Branch', 'ref_id', 'Sapience', 'name', 'Sapience'))`
	5. Setup SQL Server (from Repo: canopy-sql)
		1. Run DDL in canopy-sql/ddl
		2. Run DML in canopy-sql/dml

5. Retrieve Keys
	1. Service Bus
		1. From Azure Portal -> Service Bus -> Click on sapience-dev -> 'Shared access policies' -> 'RootManageSharedAccessKey'
		2. Copy the Primary Key
		3. In environment.dev.tfvars, set canopy_amqp_password to the Primary Key
	2. Event Hubs
		1. From Azure Portal -> Event Hubs -> Click on sapience-event-hub-journal-dev -> 'Shared access policies' -> 'RootManageSharedAccessKey'
		2. Copy the Primary Key
		3. In environment.dev.tfvars, set canopy_event_hub_password to the Primary Key
	3. Cosmos DB
		1. From Azure Portal -> Event Hubs -> Click on sapience-canopy-hierarchy-dev -> Keys
		2. Copy the Primary Key
		3. In environment.dev.tfvars, set canopy_hierarchy_cosmos_password to the Primary Key

6. Setup Canopy
	1. Deploy CronJob(s)
		1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
		2. Edit "terraform/environments/dev/cronjob/main.tf"
			- Change 'key' in terraform{} block: "sapience.environment.<font color="red">dev</font>.cronjob.terraform.tfstate"
		3. Terraform Initialize and Apply
			```
			cd terraform/environments/dev/cronjob
			terraform init -backend-config="../../../config/backend.config"
			terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
			```
		4. To make sure the secret for the "Setup Canopy" step is created, manually trigger this through the Kubernetes dashboard
	
	2. Deploy Canopy containers
		1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
	    2. Edit "terraform/environments/dev/canopy/main.tf"
		    1. Edit "terraform { backend {} }" as needed
			2. Update kubernetes_namespace_default_token in the environment.<font color="red">dev</font>.tfvars file in the config folder.  This token is automatically generated.
				1. In the Kubernetes Portal, switch to the current environment Namespace
				2. Click on Secrets and find the Secret named default-token-XXXXX
				3. Copy the characters in place of the XXXXX and replace it in the environment.<font color="red">dev</font>.tfvars file.
		3. Terraform Initialize and Apply
			```
			cd /c/projects-sapience/terraform/environments/dev/canopy
			terraform init -backend-config="../../../config/backend.config"
			terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
			```

7. Setup DNS Zone
    1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
	2. Edit "terraform/environments/dev/dns/main.tf"
		- Change 'key' in terraform{} block: "sapience.environment.<font color="red">dev</font>.dns.terraform.tfstate"
	3. Terraform Initialize and Apply
		```
		cd terraform/environments/dev/dns
		terraform init -backend-config="../../../config/backend.config"
		terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
		```
	4. You will need to manually add NS records if the DNS is hosted by another provider.  You don't need to do this step if the Zone is hosted by Azure in the same Subscription.
		1. In Azure, go to DNS Zones and click on the newly created Zone.
		2. Copy the 4 Name Servers listed.
		3. At the DNS host, create the NS records.  _(Today, this is in Net4India)_

8. Setup Databricks
    1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
	2. Edit "terraform/environments/dev/databricks/main.tf"
		- Change 'key' in terraform{} block: "sapience.environment.<font color="red">dev</font>.databricks.terraform.tfstate"
	3. Terraform Initialize and Apply
		```
		cd terraform/environments/dev/databricks
    	terraform init -backend-config="../../../config/backend.config"
		terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
		```

9. Setup Ambassador
    1. Remove any existing ".terraform" folder if copying from an existing folder and this is new non-existing infrastructure
	2. Edit "terraform/environments/dev/ambassador/main.tf"
		- Change 'key' in terraform{} block: "sapience.environment.<font color="red">dev</font>.ambassador.terraform.tfstate"
	3. Terraform Initialize and Apply
		```
		cd terraform/environments/dev/ambassador
    	terraform init -backend-config="../../../config/backend.config"
		terraform apply -var-file="../../../config/realm.lab.tfvars" -var-file="../../../config/environment.dev.tfvars"
		```
	4. After the pods are running, you will need to update the LetsEncrypt domain and token in the environment.<font color="red">dev</font>.tfvars file
		1. In the Kubernetes Portal, find the Cert Manager pod and view its logs.  Near the top, you'll find a line containing the domain and token.
			
			`"Looking up Ingresses for selector certmanager.k8s.io/acme-http-domain=1937180995,certmanager.k8s.io/acme-http-token=869731899"`
		2. Update the following settings in environment.<font color="red">dev</font>.tfvars
			- ambassador_letsencrypt_acme_http_domain
			- ambassador_letsencrypt_acme_http_token


