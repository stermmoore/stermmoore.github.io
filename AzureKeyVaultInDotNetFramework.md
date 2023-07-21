# Configuring a .net framework application running in IIS using Azure KeyVault 

When working on an asp.net framework application I had a requirement to pull in configuration from Azure KeyVault. The application was running in IIS on premise and, ideally, I didn't want to make any code changes. Most of the documentation I had found dealt with running applications in Azure under a managed identity and I ended up having to read the source code for the Configuration Builder before figuring out how to do what I needed to do. So for future reference (and in case it helps anyone else), here is the method I used...


## Setting up the Azure stuff

In Azure we need to create an app registration with a client secret, create the Key Vault and grant the app access to the Key Vault. If you're already familiar with this aspect, feel free to skip this section. For each of the stages I've included instructions for the Azure portal and the azure cli commands as an alternative. At the end of it we will have the following bits of information:

- Tenant ID
- Client (Application) ID
- Client Secret
- Key Vault URI

### App registration - In this step we create the credentials that the app will use to connect to Azure

- In the Azure portal, go to Active Directory and click the App Registrations blade.
- Click on New Registration, enter a name for your app, then click Register.
- You should now see a screen containing the 'Application (client) ID' and 'Directory (tenant) ID'; make a note of these.

```powershell
az login --tenant [your tenant ID]

az ad app create --display-name [name of the app you want to create]

az ad sp create --id [ID of the app you just created - this will have been returned by the above command] 
```

### Generate a client secret (the password that your app will use to connect to Azure)

- Click on Certificates and Secrets on the Manage section of the App Registration
- Highlight the Client Secrets tab and click Add Client Secret
- Add a description and set the expiry date, then click Add
- You will then be shown the client secret value; make a note of this (you won't be able to access it again)

```powershell
az ad app credential reset --id [app id] --display-name [will appear in the description field in the portal] #this command returns the client secret in the "password" field
```

### Create the Key Vault
- Search for Key Vault in Azure and click Create
- Select your Subscription and Resource Group
- Enter a name for the Key Vault (this must be globally unique, so don't waste good names)
- Select the Region and Tier, then click Review + Create, then Create (once reviewed)
- Make a note of the Key Vault URI

```powershell
az keyvault create --name [globally unique Key Vault Name] --resource-group [Resource Group] --location [Location e.g. UKSouth] #this will return an ID which will be used in the next step
```

### Grant yourself access to the Key Vault

By default you will not have access to Secrets (or, indeed, anything) within the Key Vault

If your Key Vault has been set up with the Azure role-based access control permission model (visible on Overview > Properties > Access Configuration), then you can grant access using the Access Control (IAM) blade.

Giving yourself access to manage the Key Vault
- On the Access Control (IAM) blade, click the Add button and select Add Role Assignment, select Key Vault Contributor then click Next
- Select User, Group or service principal
- Click "+ Select Members" find your user account, then click Select
- Click Review and Assign

You should now have access to add Secrets.

```powershell
az role assignment create --role "Key Vault Contributor" --assignee [your user principal name] --scope /subscriptions/[Subscription ID]/resourcegroups/[Resource Group]/providers/Microsoft.KeyVault/vaults/[Key Vault Name] #The scope field should be the Key Vault ID which will have been returned by the previous command
```

### Grant your application access to the Key Vault (pretty much the same as above)

- On the Access Control (IAM) blade, click the Add button and select Add Role Assignment, select Key Vault Secrets User, then click Next
- Select User, Group or service principal
- Click "+ Select Members" find your application registration, then click Select
- Click Review and Assign
 
The app should now have access to List and Read Secrets.

```powershell
az role assignment create --role "Key Vault Secrets User" --assignee [your application ID] --scope /subscriptions/[Subscription ID]/resourcegroups/[Resource Group]/providers/Microsoft.KeyVault/vaults/[Key Vault Name]
```

### Adding a Secret

- Click on the Secrets blade, then Generate/Import
- Give the Secret a Name and Value, then click Create

```powershell
az keyvault secret set --vault-name [Key Vault Name] --name [Secret Name] --value [Secret Value]
```

## Using Key Vault Secrets in .net framework

Using the wizard:

- Right click on the Solution in Visual Studio and click Add > Connected service
- Add a Service Dependency
- Select Azure Key Vault, then click Next, then Next again, then Finish
- You may be prompted to sign into your Azure account. Make sure this is the account that you have created the Key Vault in.

This will add several NuGet Packages to your project and add a config providers section to your Web.Config containing the AzureKeyVault builder.

```xml
  <configSections>
    <section name="configBuilders" type="System.Configuration.ConfigurationBuildersSection, System.Configuration, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a" restartOnExternalChanges="false" requirePermission="false" />
  </configSections>
  <configBuilders>
    <builders>
    <add name="AzureKeyVault" vaultName="[VaultName]" type="Microsoft.Configuration.ConfigurationBuilders.AzureKeyVaultConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.Azure, Version=2.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" /></builders>
  </configBuilders>
```

Modify this section as follows: 

- Replace the vaultName attribute with uri and set the value to your Key Vault URI, e.g. uri="https://my-unique-key-vault-name.vault.azure.net/"
- In order to use values from the Key vault as appSettings, add configBuilders="AzureKeyVault" to the appSettings element.


The configBuilders section should now look like this:
```xml
  <configBuilders>
    <builders>
      <add name="AzureKeyVault" uri="https://my-unique-key-vault-name.vault.azure.net/" type="Microsoft.Configuration.ConfigurationBuilders.AzureKeyVaultConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.Azure, Version=2.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" />
    </builders>
  </configBuilders>
```

And appSettings should look like this:
```xml
  <appSettings configBuilders="AzureKeyVault">
    <add key="TestKey" value="From Web.Config" />
  </appSettings>
```

If you launch your application now, it will either work (because you are running it under an account which has access to the KeyVault), or throw the following error: "CredentialUnavailableException: DefaultAzureCredential failed to retrieve a token from the included credentials." This error means that the application cannot find the credentials required to access the KeyVault. To understand this further it is worth looking into the [DefaultAzureCredential class](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential?view=azure-dotnet). This is widely used and is therefore worthwhile understanding. 

DefaultAzureCredential tries multiple credential types in order, stopping at the first available. The first tried is [EnvironmentCredential](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.environmentcredential?view=azure-dotnet) which is the one we want to use.

For this we need to create 3 environment variables with the following names:

- AZURE_TENANT_ID
- AZURE_CLIENT_ID
- AZURE_CLIENT_SECRET

These must be available to the process that the website is running in - you may need to restart Visual Studio / IIS.

This should now work. Any variables present in your Web.Config AppSettings which have a matching name in Key Vault should be replaced. You can also use Key Vault settings within the connectionStrings section by adding the configBuilders="AzureKeyVault" attribute to the connectionStrings element.

## Deploying To Multiple Environments

In most circumstances you will be deploying to multiple environments, each with its own KeyVault, so you will need to change the uri attribute of the AzureKeyVault builder for each environment. There are a number of ways to do this, for example, you could have environment specific web.[env].config transformations or a step in your deployment pipeline to swap the setting over. Here, I'm going to look at using an environment variable.

1) Add an AppSetting called AZURE_KEY_VAULT_URI (this could be any name, but I've followed the Microsoft convention) 
2) Change the uri attribute of the AzureKeyVault builder to uri="${AZURE_KEY_VAULT_URI}" 
3) Add the NuGet package Microsoft.Configuration.ConfigurationBuilders.Environment
4) Add the Environment config builder to the configBuilders section
```xml
<add name="Environment" type="Microsoft.Configuration.ConfigurationBuilders.EnvironmentConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.Environment, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" />
```
5) Change the configBuilders attribute of the appSettings element to "Environment,AzureKeyVault"
6) Add an environment variable called AZURE_KEY_VAULT_URI containing the URI

The Key Vault used should now be goverened by the environment variable.

## Environment Variables Not Available? It could be this:

Working locally using Windows 11 Pro, I found that user environment variables were not available to the IIS process. After some investigation I found the setProfileEnvironment application pool setting, which governs the environment variables available to the application process. The documentation states that this defaults to true, meaning user environment variables should be available, but I found this not to be the case for some operating systems. 

I haven't found a way to change this via IIS Manager but editing the applicationHost.config file seems to work:

1) Open the applicationHost.config file (located in %windir%\system32\inetsrv\config)
2) Find the applicationPools element
3) Find the add element for the applicationPool for your site (or the applicationPoolDefaults element if you want to make user variables available as the default)
4) Under the app pool (or default) element there should be a processModel element, this will contain the user that you have specified the app pool to run as 
5) Add or update the following attributes within the processModel element loadUserProfile="true" setProfileEnvironment="true", e.g. :

```xml
<add name="MyAppPool">
  <processModel identityType="SpecificUser" userName=".\MyAppPoolUser" password="[Redacted]" loadUserProfile="true" setProfileEnvironment="true" />
</add>

```
Changing this setting should result in the user's environment variables being available to the web site process.


### When developing locally

If using environment variables they must be available to the Visual Studio process. If not, then the other means of authentication will be used. 


## References


This post got me started with adding KeyVault to the application 

https://dotnetthoughts.net/azure-key-vault-in-aspnet-mvc/

Microsoft documentation relating to the IIS Application Pool setProfileEnvironment setting 

https://learn.microsoft.com/en-us/iis/configuration/system.applicationhost/applicationpools/add/processmodel

Source code for the AzureKeyVaultConfigurationBuilder showing the use of the DefaultAzureCredential class 

https://github.com/aspnet/MicrosoftConfigurationBuilders/blob/main/src/Azure/AzureKeyVaultConfigBuilder.cs#L166

Microsoft documentation for the DefaultAzureCredential class

 https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential?view=azure-dotnet

Microsoft documentation for using environment credentials

 https://learn.microsoft.com/en-us/dotnet/api/azure.identity.environmentcredential?view=azure-dotnet