---
layout: post
title:  "Using Azure Key Vault in ASP.NET"
date:   2019-03-18 06:00:00 +0200
---

*Note: This tutorial heavily depends on the configuration mechanism provided by [configuration builders][config-builders] what is only available in .NET Framework 4.7.1 and later. The tutorial also assumes that you already host your web application on Azure App Service.*

Key Vault is a very handy Azure service which you can use to avoid storing credentials in your code and in your version control system, thus reducing the chance of those secrets being compromised. Instead of storing for example a connection string in your `Web.config` (in case of ASP.NET), you can store that credential in this cloud service which your app can authenticate against with Azure Active Directory, have role-based access control and use many more of its great features. Please refer to [the official documentation of Key Vault][key-vault-overview] for further information.

It's quite trivial to integrate Azure Key Vault with an ASP.NET Core application running on Azure App Service, because there are tons of documentation and examples out there. On the other hand, doing the same with classic ASP.NET (targeting the full .NET Framework, not .NET Core), the resources are a bit sparser and scattered and thus it requires considerably more figuring out on the side of the developer. That's why I'd like to give here an easy to follow, step-by-step tutorial about doing the latter, while pointing out the potential pitfalls during the process.

If you are working with Key Vault and Azure App Service, I also highly recommend [this repo][github-ms-sample] to take a look at. Here you can read more about the authentication aspects of this scenario and how that exactly works under the hood, while my tutorial below concentrates more on the ASP.NET-specific things.

## The goal of this tutorial

The whole premise of this tutorial is to show you how to take an existing ASP.NET application running in Azure App Service and without touching the C# code, configure it in a way that it will retrieve its sensitive configuration values from a Key Vault instead of its `Web.config` file. So when you'd write for example

```csharp
ConfigurationManager.AppSettings["StorageConnectionString"]
```

somewhere in your application, that value would come directly from a secret in Key Vault. All without modifying anything in the actual source code of your app.

I will also describe a more advanced scenario which will make your Web App integrated with Key Vault very easily redeployable into new environments.

## Create the Key Vault instance

First, you'll need a Key Vault instance with secrets in it which you will use to retrieve your config values from in your application. You can follow [the official docs][key-vault-create] here, there's no big magic, it's all very simple.

## Add secrets to the Key Vault

Before you can access them from your code, you need to add those secrets to the Key Vault. In order to do that, follow [these steps][key-vault-add-secret]. It's important that the secrets must have the same name as the app settings you would like to replace with them. So for example if you have an app setting with the key `StorageConnectionString` in your `Web.config` file, you must create a secret with the same name and the same value in Key Vault.

## Give access to your Web App to the secrets stored in Key Vault

The tricky part comes when you'd like to access those secrets from your ASP.NET application.
Before actually touching the code, you need to make sure that the Web App which will host your application is able to authenticate against your Key Vault. In order to do that, it's best to use the Web App's [managed identity][managed-identity] feature.

In a nutshell, this feature will provide your Web App with its own identity in your Azure Active Directory and your application will be able to authenticate with Key Vault using that identity. The benefit of this is that the only information you'll need to pass to your application is the name of your Key Vault instance to connect to. This way, you will be able to keep ALL sensitive data in Key Vault, since you won't need to store any credentials to connect to the Key Vault itself, everything is handled by Azure. More info on this topic can be found [here][key-vault-managed-identity].

After you've set up managed identity for your Web App, you need to tell your Azure Key Vault to let that application access its secrets. If you do that via the Azure Portal, you need to open your Key Vault instance and do the configuration on the **Access policies** blade. In this case there are two pitfalls (both reported in [this GitHub issue][github-issue]) that you must make sure to avoid if you don't want to scratch your head later while thinking about why nothing works while you did everything (supposedly) right.

[First][github-issue-1st], make sure you only set the **Select principal** field to your Web App's managed identity and leave the **Authorized application** field blank.

[Second][github-issue-2nd], if you get two results when searching for your principal's name, select the one with the icon depicting a small cloud in front of four rectangles from the two options.

## Integrate Key Vault in code

Now we can finally get to the code! You need to whip out Visual Studio, open your ASP.NET project and if you haven't done yet, this is the best time to update your app to target .NET Framework 4.7.1 or later. It's because the rest of the tutorial heavily depends on the presence of configuration builders and they are only available in recent versions of the framework.

Configuration builders in ASP.NET have the same goal as chaining [configuration providers][config-providers] in the setup code in ASP.NET Core. They provide a way for you to modify and/or override the values coming from your configuration files (`Web.config` in case of ASP.NET, `appsettings.json` in case of ASP.NET Core) by using different sources (environment variables, Key Vault, etc.). This means that you can store your config values in other places than those files without modifying the parts of your application that are dependent on those values. You can read more about configuration builders [here][config-builders].

### Simple configuration

When/if your project already targets the right version of the framework, go ahead and [add Key Vault as a connected service][key-vault-connected-service] to it. This will add a couple of NuGet packages to your project and a configuration builder - pointing to the Key Vault instance you've chosen during the setup - to your `Web.config` file.

Next, remove the `vaultUri` attribute of the freshly added Key Vault builder, so it would look something like this: (Obviously with your Key Vault instance's name in the `vaultName` attribute.)

```xml
<add name="AzureKeyVault" vaultName="your vault's name" type="Microsoft.Configuration.ConfigurationBuilders.AzureKeyVaultConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.Azure, Version=1.0.0.0, Culture=neutral" />
```

Then replace the `appSettings` tag in your `Web.config` with this:

```xml
<appSettings configBuilders="AzureKeyVault">
```

Now if you deploy your application to the Azure Web App which you gave access to the Key Vault previously, it will all work like a charm. You will be able to access secrets from Key Vault in your code as if they were good old app settings in your `Web.config` file.

### Debugging locally

You might have noticed that at the moment the only credential your application knows about the Key Vault instance it should connect to is its name or its URI. (In the above example I kept the name, but its up to you which one you choose.) No ID-s, no secrets, nothing. Then how will the app authenticate to Key Vault to access its secrets? If you deploy your application to an Azure Web App, the question has already been answered earlier in this tutorial: using your app's managed identity.

But to be able debug your application locally with Visual Studio, you'll need to setup one more thing. The thing you need is [this extension][vs-auth-extension] for older version of Visual Studio - the newer releases contain this feature by default. Nevertheless, you need to follow the steps outlined in its description to authenticate with your Azure account in VS. After that you need to add that very same account to Key Vault, exactly the same way as you did with the service principal earlier. (If you used the same account for provisioning the Key Vault, it will be already there.) After this, whenever you debug your application locally, it will authenticate against Key Vault using your own Azure account and your app will be able to access the secrets just like when it is running on App Service. You can read about how this all works under the hood [here][github-ms-sample].

### Advanced configuration

So far everything is in Key Vault and you can even debug your app locally. But there's one ugly piece left: the Key Vault's name itself is still hardcoded in the `Web.config` file. If you only plan to deploy your app to one particular environment, it might not be a problem, but in the world of Agile, DevOps and CI/CD, I highly doubt that you would like to keep it that way. Of course you can use some XML transformation as part of your CI/CD pipeline to replace the name of the Key Vault before each deployment, but I have an even more elegant solution for this.

First, update the

```
Microsoft.Configuration.ConfigurationBuilders.Azure
```

NuGet package to 2.0.0-beta. Even though this is (as it's name indicates) a beta version at the time of writing, I already used it extensively and haven't experienced any issues. You also need to install the same version of the

```
Microsoft.Configuration.ConfigurationBuilders.Environment
```

NuGet package.
Installing these will reset your previous changes to the `configBuilders` section of your `Web.config`, so go ahead and configure it again, but this time like this:

```xml
<configBuilders>
  <builders>
    <add name="Environment" type="Microsoft.Configuration.ConfigurationBuilders.EnvironmentConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.Environment, Version=2.0.0.0, Culture=neutral" />
    <add name="AzureKeyVault" vaultName="${KEY_VAULT_NAME}" type="Microsoft.Configuration.ConfigurationBuilders.AzureKeyVaultConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.Azure, Version=2.0.0.0, Culture=neutral" />
  </builders>
</configBuilders>
```

Also modify your `appSettings` tag to look like this:

```xml
<appSettings configBuilders="Environment,AzureKeyVault">
  <add key="KEY_VAULT_NAME" value=""/>
</appSettings>
```

The only thing left to do now is to set the `KEY_VAULT_NAME` environment variable's value. On your local machine, you can set an actual environment variable while debugging. On the Azure Web App, which hosts your application, you'll need to [add an app setting][app-service-settings] with it's key being `KEY_VAULT_NAME` and it's value being the name of your Key Vault. It's because Azure App Service automatically adds a new environment variable for each app setting you configure on the portal. This is the exact behavior we will take advantage of in this scenario.

So what happens in this case is that at runtime, even before your app runs, the App Service will turn the `KEY_VAULT_NAME` from the Web App's app settings into an environment variable as described above. Then on application startup, the newly addedd `Environment` config builder will run, which will fill the value of the app setting named `KEY_VAULT_NAME` from the environment variable with the very same name. Then the `AzureKeyVault` config builder runs, which will get it's `vaultName` attribute from the above mentioned app setting. Finally the `AzureKeyVault` config builder acts exactly as before and it will pull all the secrets from that particular Key Vault instance into the app's settings. Phew!

For more details about how these all works under the hood, please refer to the [official docs][config-builder-params].

## Automated deployment

One obvious benefit of the above discussed advanced configuration is that it makes extremely easy to deploy the infrastructure of your application to different Azure environments.

[Here][github-sample] you can find a [sample application][github-sample-application] that contains the implementation of the above detailed steps **plus** an [ARM template][github-arm] that you can use as a basis of your own deployments. So if you want to see everything we discussed so far in motion, just deploy the ARM template to Azure, publish the sample application into the freshly created Web App, send an HTTP GET request to it's /api/values endpoint and in the response you will be able to see the value of the secret you've saved in the Key Vault during the ARM deployment.

Since you are reading this blog post, I will assume that you are familiar with ARM templates - and if you are not, [this][arm-quickstart] is a great starting point - and I won't explain the whole template. But I would still like to point out some interesting bits and pieces what are responsible for facilitating the connection between the Web App and the Key Vault. (Thus some parts are removed for brevity below.)

Here, in the Web App's definition, we are telling the Azure Resource Manager to create a managed identity for the app and also set the `KEY_VAULT_NAME` app setting to the name of the soon to be created Key Vault:

```json
{
  "name": "[parameters('webAppName')]",
  "type": "Microsoft.Web/sites",
  "identity": {
    "type": "SystemAssigned"
  },
  "properties": {
    "siteConfig": {
      "appSettings": [
        {
          "name": "KEY_VAULT_NAME",
          "value": "[parameters('keyVaultName')]"
        }
      ]
    }
  }
}
```

Then, when we create the Key Vault, we add that identity to it's access policies:

```json
{
  "name": "[parameters('keyVaultName')]",
  "type": "Microsoft.KeyVault/vaults",
  "properties": {
    "accessPolicies": [
      {
        "tenantId": "[reference(variables('identityResourceId'), '2015-08-31-PREVIEW').tenantId]",
        "objectId": "[reference(variables('identityResourceId'), '2015-08-31-PREVIEW').principalId]",
        "permissions": {
          "secrets": [
            "get"
          ]
        }
      }
    ]
  }
}
```

## Conclusion

This is the closest solution I found to mimic configuration providers from ASP.NET Core in ASP.NET. This is just one possible way of using config builders in ASP.NET and I think this tutorial shows perfectly the flexibility of this relatively small piece of the framework.

[key-vault-overview]: https://docs.microsoft.com/en-us/azure/key-vault/key-vault-overview
[github-ms-sample]: https://github.com/Azure-Samples/app-service-msi-keyvault-dotnet
[key-vault-create]: https://docs.microsoft.com/en-us/azure/key-vault/quick-create-portal
[key-vault-add-secret]: https://docs.microsoft.com/en-us/azure/key-vault/quick-create-portal
[managed-identity]: https://docs.microsoft.com/en-us/azure/app-service/overview-managed-identity
[key-vault-managed-identity]: https://docs.microsoft.com/en-us/azure/key-vault/tutorial-net-create-vault-azure-web-app#managed-service-identity-and-how-it-works
[github-issue]: https://github.com/Azure/azure-sdk-for-net/issues/4190
[github-issue-1st]: https://github.com/Azure/azure-sdk-for-net/issues/4190#issuecomment-378955080
[github-issue-2nd]: https://github.com/Azure/azure-sdk-for-net/issues/4190#issuecomment-395235255
[config-providers]: https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.2#providers
[config-builders]: https://docs.microsoft.com/en-us/aspnet/config-builder
[key-vault-connected-service]: https://docs.microsoft.com/en-us/azure/key-vault/vs-key-vault-add-connected-service#add-key-vault-support-to-your-project
[vs-auth-extension]: https://marketplace.visualstudio.com/items?itemName=chrismann.MicrosoftVisualStudioAsalExtension
[app-service-settings]: https://docs.microsoft.com/en-us/azure/app-service/web-sites-configure
[config-builder-params]: https://github.com/aspnet/MicrosoftConfigurationBuilders#appsettings-parameters
[config-builder-app-service]: https://github.com/aspnet/MicrosoftConfigurationBuilders#azureappconfigurationbuilder
[github-sample]: https://github.com/peterbozso/key-vault-config-builder-sample
[github-sample-application]: https://github.com/peterbozso/key-vault-config-builder-sample/tree/master/src
[github-arm]: https://github.com/peterbozso/key-vault-config-builder-sample/blob/master/azuredeploy.json
[arm-quickstart]: https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-quickstart-create-templates-use-the-portal