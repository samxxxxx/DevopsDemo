# DevopsDemo

# Azure 终端登录

以下设置为手动通过 Azure CLI 部署 Aspnet web app 的方法，参考[链接](https://docs.azure.cn/zh-cn/app-service/deploy-github-actions?tabs=openid%2Caspnetcore)

身份验证，使用 OpenID Connect

首先在 Azure 终端，尝试登录到 Microsoft Azure CLI 

```
az login --scope https://graph.microsoft.com//.default
```
要求在浏览器打开链接 `https://microsoft.com/devicelogin` 后输入验证码，并选择登录。输入代码以允许访问 `SP9LBEMM2`

```
Cloud Shell is automatically authenticated under the initial account signed-in with. Run 'az login' only if you need to use a different account
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code SP9LBEMM2 to authenticate.
```

# 1. 创建 Active Directory 应用程序

```bash
az ad app create --display-name DevopsDemoApp
```

此命令将输出 JSON，其中 `appId` 为你的 `client-id`。 保存该值，稍后将其用作 `AZURE_CLIENT_ID` GitHub 机密。

<details>
<summary>JSON 输出</summary>

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#applications/$entity",
  "addIns": [],
  "api": {
    "acceptMappedClaims": null,
    "knownClientApplications": [],
    "oauth2PermissionScopes": [],
    "preAuthorizedApplications": [],
    "requestedAccessTokenVersion": null
  },
  "appId": "3aa94c7d-a89b-46c0-90b0-dcde9a16874a",
  "appRoles": [],
  "applicationTemplateId": null,
  "certification": null,
  "createdDateTime": "2025-03-09T07:10:59.5742409Z",
  "defaultRedirectUri": null,
  "deletedDateTime": null,
  "description": null,
  "disabledByMicrosoftStatus": null,
  "displayName": "DevopsDemoApp",
  "groupMembershipClaims": null,
  "id": "d716f00d-e85d-4b87-930e-35361d1e3823",
  "identifierUris": [],
  "info": {
    "logoUrl": null,
    "marketingUrl": null,
    "privacyStatementUrl": null,
    "supportUrl": null,
    "termsOfServiceUrl": null
  },
  "isDeviceOnlyAuthSupported": null,
  "isFallbackPublicClient": null,
  "keyCredentials": [],
  "nativeAuthenticationApisEnabled": null,
  "notes": null,
  "optionalClaims": null,
  "parentalControlSettings": {
    "countriesBlockedForMinors": [],
    "legalAgeGroupRule": "Allow"
  },
  "passwordCredentials": [],
  "publicClient": {
    "redirectUris": []
  },
  "publisherDomain": "samx5hotmail.onmicrosoft.com",
  "requestSignatureVerification": null,
  "requiredResourceAccess": [],
  "samlMetadataUrl": null,
  "serviceManagementReference": null,
  "servicePrincipalLockConfiguration": null,
  "signInAudience": "AzureADMyOrg",
  "spa": {
    "redirectUris": []
  },
  "tags": [],
  "tokenEncryptionKeyId": null,
  "uniqueName": null,
  "verifiedPublisher": {
    "addedDateTime": null,
    "displayName": null,
    "verifiedPublisherId": null
  },
  "web": {
    "homePageUrl": null,
    "implicitGrantSettings": {
      "enableAccessTokenIssuance": false,
      "enableIdTokenIssuance": false
    },
    "logoutUrl": null,
    "redirectUriSettings": [],
    "redirectUris": []
  }
}
```
</details>

运行成功后，会在 Microsoft Entra Id 中会创建一个名为 DevopsDemoApp 的应用程序。

# 2. 创建服务主体。 

将 `$appID` 替换为 JSON 输出中的 `appId`。

```bash
az ad sp create --id $appID

az ad sp create --id 3aa94c7d-a89b-46c0-90b0-dcde9a16874a
```

复制 `appOwnerOrganizationId` 以在稍后用作 `AZURE_TENANT_ID` 的 GitHub 机密。

<details>
<summary>JSON 输出</summary>

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#servicePrincipals/$entity",
  "accountEnabled": true,
  "addIns": [],
  "alternativeNames": [],
  "appDescription": null,
  "appDisplayName": "DevopsDemoApp",
  "appId": "3aa94c7d-a89b-46c0-90b0-dcde9a16874a",
  "appOwnerOrganizationId": "57f14ae6-fb55-4c5b-8f3b-a27c540b9cf7",
  "appRoleAssignmentRequired": false,
  "appRoles": [],
  "applicationTemplateId": null,
  "createdDateTime": "2025-03-09T07:19:57Z",
  "deletedDateTime": null,
  "description": null,
  "disabledByMicrosoftStatus": null,
  "displayName": "DevopsDemoApp",
  "homepage": null,
  "id": "98b12728-9052-41bf-823d-8eb4cdf408f6",
  "info": {
    "logoUrl": null,
    "marketingUrl": null,
    "privacyStatementUrl": null,
    "supportUrl": null,
    "termsOfServiceUrl": null
  },
  "keyCredentials": [],
  "loginUrl": null,
  "logoutUrl": null,
  "notes": null,
  "notificationEmailAddresses": [],
  "oauth2PermissionScopes": [],
  "passwordCredentials": [],
  "preferredSingleSignOnMode": null,
  "preferredTokenSigningKeyThumbprint": null,
  "replyUrls": [],
  "resourceSpecificApplicationPermissions": [],
  "samlSingleSignOnSettings": null,
  "servicePrincipalNames": [
    "3aa94c7d-a89b-46c0-90b0-dcde9a16874a"
  ],
  "servicePrincipalType": "Application",
  "signInAudience": "AzureADMyOrg",
  "tags": [],
  "tokenEncryptionKeyId": null,
  "verifiedPublisher": {
    "addedDateTime": null,
    "displayName": null,
    "verifiedPublisherId": null
  }
}
```
</details>

# 3. 按订阅和对象创建新的角色分配。 

默认情况下，角色分配将绑定到默认订阅。 将 `$subscriptionId` 替换为你的订阅 `ID`，将 `$resourceGroupName` 替换为你的资源组名称，将 `$webappName` 替换为你的 Web 应用名称，将 `$assigneeObjectId` 替换为生成的 id。

```bash
az role assignment create --role contributor --subscription $subscriptionId --assignee-object-id  $assigneeObjectId --scope /subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.Web/sites/$webappName --assignee-principal-type ServicePrincipal

az role assignment create --role contributor --subscription 35686e51-5090-4d44-83cc-ca31aa6876c5 --assignee-object-id  98b12728-9052-41bf-823d-8eb4cdf408f6 --scope /subscriptions/35686e51-5090-4d44-83cc-ca31aa6876c5/resourceGroups/CICD_group/providers/Microsoft.Web/sites/SamDevopsWebApp --assignee-principal-type ServicePrincipal

```

<details>
<summary>JSON 输出</summary>

```json
{
  "condition": null,
  "conditionVersion": null,
  "createdBy": null,
  "createdOn": "2025-03-09T07:42:54.624090+00:00",
  "delegatedManagedIdentityResourceId": null,
  "description": null,
  "id": "/subscriptions/35686e51-5090-4d44-83cc-ca31aa6876c5/resourceGroups/CICD_group/providers/Microsoft.Web/sites/SamDevopsWebApp/providers/Microsoft.Authorization/roleAssignments/6a76bc79-a08a-4e2d-99e2-f94b2ce4c796",
  "name": "6a76bc79-a08a-4e2d-99e2-f94b2ce4c796",
  "principalId": "98b12728-9052-41bf-823d-8eb4cdf408f6",
  "principalType": "ServicePrincipal",
  "resourceGroup": "CICD_group",
  "roleDefinitionId": "/subscriptions/35686e51-5090-4d44-83cc-ca31aa6876c5/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c",
  "scope": "/subscriptions/35686e51-5090-4d44-83cc-ca31aa6876c5/resourceGroups/CICD_group/providers/Microsoft.Web/sites/SamDevopsWebApp",
  "type": "Microsoft.Authorization/roleAssignments",
  "updatedBy": "ec283b6f-bf32-4fa6-8a64-e8a4631806d5",
  "updatedOn": "2025-03-09T07:42:55.147656+00:00"
}

```
</details>

# 4. 创建新的联合标识凭据

- 将 `APPLICATION-OBJECT-ID` 替换为 第 1 步创建返回的应用程序的 `appId`（在创建应用时生成）。
- 为 `CREDENTIAL-NAME` 设置一个值供以后引用。
- 设置 `subject` ：`repo:organization/repository:ref:refs/heads/main`
  - `organization` 替换为 你的 GitHub 存储库所有者。
  - `repository` 替换为 你的 GitHub 存储库名称。
  - `main` 替换为 你的 GitHub 存储库分支名称。

`subject` 的设置有 3 种形式：

- GitHub Actions 环境中的作业：repo:< Organization/Repository >:environment:< Name >
- 对于未绑定到环境的作业，请根据用于触发工作流的 ref 路径包括分支/标记的 ref 路径：repo:< Organization/Repository >:ref:< ref path>。 例如 repo:n-username/ node_express:ref:refs/heads/my-branch 或 repo:n-username/ node_express:ref:refs/tags/my-tag。
- 对于由拉取请求事件触发的工作流：repo:< Organization/Repository >:pull_request。

将修改后的 JSON 保存为 `credential.json` 文件。

**要注意 `issuer` 的结尾不能以 / 结尾，否则会造成 Azure CLI 登录命令执行失败**
**`issure` 必须是 `https://token.actions.githubusercontent.com` 而不能是 `https://token.actions.githubusercontent.com/` 。**

```bash
az ad app federated-credential create --id <APPLICATION-OBJECT-ID> --parameters credential.json
("credential.json" contains the following content)
{
    "name": "<CREDENTIAL-NAME>",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:organization/repository:ref:refs/heads/main",
    "description": "Testing",
    "audiences": [
        "api://AzureADTokenExchange"
    ]
}

# 拼接结果：
az ad app federated-credential create --id 3aa94c7d-a89b-46c0-90b0-dcde9a16874a --parameters '{ "name": "DevopsDemoCREDENTIAL", "issuer": "https://token.actions.githubusercontent.com", "subject": "repo:samxxxxx/DevopsDemo:ref:refs/heads/main", "description": "设置 GitHub 登录的 Azure 联合凭据", "audiences": [ "api://AzureADTokenExchange" ] }'

```

返回的 JSON：

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#applications('d716f00d-e85d-4b87-930e-35361d1e3823')/federatedIdentityCredentials/$entity",
  "audiences": [
    "api://AzureADTokenExchange"
  ],
  "description": "设置 GitHub 登录的 Azure 联合凭据",
  "id": "9712db84-fddd-4659-8707-0fc63a3e7cdf",
  "issuer": "https://token.actions.githubusercontent.com/",
  "name": "DevopsDemoCREDENTIAL",
  "subject": "repo:samxxxxx/DevopsDemo:ref:refs/heads/main"
}
```

删除联合标识凭据

--id
应用程序的 appId、identifierUri 或 ID（以前称为 objectId）。

--federated-credential-id
联合标识凭据的 ID 或名称。

```bash
az ad app federated-credential delete --id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx --federated-credential-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
az ad app federated-credential delete --id 3aa94c7d-a89b-46c0-90b0-dcde9a16874a --federated-credential-id 9712db84-fddd-4659-8707-0fc63a3e7cdf
```

# 5. 在 GitHub 中创建3个 secrets

- `AZURE_CLIENT_ID`：应用 ID，即 `appId`。
- `AZURE_TENANT_ID`：租户 ID，即 `appOwnerOrganizationId`。
- `AZURE_SUBSCRIPTION_ID`：订阅 ID，即 `subscriptionId`。

# 6. 将工作流文件添加到 GitHub 存储库

<details>
<summary>工作流文件</summary>

```yaml
name: 发布 .NET Core Web App

on: [push]

permissions:
      id-token: write
      contents: read

env:
  AZURE_WEBAPP_NAME: DevopsDemo    # set this to your application's name
  AZURE_WEBAPP_PACKAGE_PATH: '.'      # set this to the path to your web app project, defaults to the repository root
  DOTNET_VERSION: '9.0.x'           # set this to the dot net version to use

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # Checkout the repo
      - name: Checkout 代码及登录
      - uses: actions/checkout@main
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # Setup .NET Core SDK
      - name: Setup .NET Core 设置 .NET Core SDK
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }} 
    
      # Run dotnet build and publish
      - name: dotnet build and publish 发布
        run: |
          dotnet restore
          dotnet build --configuration Release
          dotnet publish -c Release --property:PublishDir='${{ env.AZURE_WEBAPP_PACKAGE_PATH }}/publish' 
        
      # Deploy to Azure Web apps
      - name: 'Run Azure webapp deploy action using publish profile credentials'
        uses: azure/webapps-deploy@v3
        with: 
          app-name: ${{ env.AZURE_WEBAPP_NAME }} # Replace with your app name
          package: '${{ env.AZURE_WEBAPP_PACKAGE_PATH }}/publish'
    
      - name: logout
        run: |
          az logout
```

</details>