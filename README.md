
# Helpdesk ID prove request

Follow the steps in this article to deploy the Helpdesk IF prove solution. For the solution overview, please check the [Helpdesk ID prove request overview](Overview.md) document.

## 1. Register an application

To begin, register your application in Microsoft Entra ID. This registration allows your backend web APIs (Azure Logic Apps) to authenticate securely using credentials like client secrets or certificates. This process ensures that only authorized applications (like the Logic App) can request an access token and interact with the Verified ID APIs.

1. Sign in to the Microsoft Entra admin center with appropriate administrator permissions.
1. From the menu, select **applications** and then **App registrations**.
1. From the list of applications, select **+ New registration**.
    1. Enter a display **Name**. For example: “Verified ID app”.
    1. For the **Supported account types**, select `Accounts in this organizational directory only`.
    1. Select **Register** to create the application.
1. After the application successfully registered, from the **overview** page, copy the “Application ID”.
1. While in the **overview** page, take a note of the **tenant ID**.
1. Next, you grant permissions to the verified ID Request Service principal. To do so: 
    1. From the menu, select **API permissions**.
    1. Select **Add a permission**.
    1. Select **APIs my organization uses**.
    1. Search for the `Verifiable Credentials` and select the `Verifiable Credentials Service Request`.
    1. Choose **Application Permission** and select the `VerifiableCredential.Create.PresentRequest` option. It allows the application to create verified ID presentation requests.
    1. Finally, select **Add permissions**.
1. Admin consent is required for this permission. So, select **Grant admin consent** for your tenant.
1. To ensure that only your application can request and receive security tokens to access the “verified ID service request API”, it is necessary to add a credential.
    1. From the main menu, select &**Certificates & secrets**.
    1. Select **New client secret**.
    1. Enter a **description** for the client secret.
    1. Under **Expires**, select a duration for which the secret is valid.
    1. Then select **Add**.
    1. Record the **secret's Value**. You'll use this value for configuration in a later step. The secret’s value won't be displayed again and isn't retrievable by any other means.

## 2. Get the authority of your verified ID service
 
1. From the menu, under **verified ID**, select **credentials**.
1. Select one of the credentials, like the **Verified employee**.
1. From the **request body** JSON, copy the value of the **authority** ID. The authority is your Decentralized Identifier. The authority ID is required even if you have already copied the tenant ID that uniquely identifies your tenant. This is because your verified ID service can verify credentials issued by both your organization and other organizations. Therefore, it is necessary to specify which credentials, such as verified employees, users can present.

## 3. Create Azure Logic app service

1. Go to portal.azure.com and sign in with your admin account.
1. Search and select **Logic apps**.
1. On the Logic apps page, select **Add**.
1. Select one of the **Standard hosting** options.
1. On the **Basics** tab:
    1. Select your **Azure subscription**.
    2. Select a **Resource group** where the logic app and its related resources will be created. You can also create a new one.
1. Enter a **Name** for your logic app. This name must be globally unique.
1. Choose a **Region** for your logic app.
1. Then a **Service plan** that defines a set of compute resources for the Logic App to run. 
1. Finally, select the **pricing plan**. This choice can be changed later. 
1. More settings are available to you. However, you can leave the default settings and select **review and create**.
1. After Azure validates your logic app setting, select **create**.
1. On the deployment completion page, you can select **Go to resource**. Or select the logic app from the search box.

## 4. Prepare the Blob Storage account

When an Azure Logic App (Standard) is created, it comes with a Storage account. The storage account is used for workflows involving state management, file handling, or integration with other Azure services. The  storage account can be utilized to host a state table that monitors the progress of verified ID presentation requests.

1. In the Azure Logic, under  **Settings**, select **Identity**
1. In the **system identity** tab, make sure the **Status** is **On**.
2. Next assign the **system identity** principal permission to access the storage account. To do so 
    1. Open the underling storage account (you can find it in the resource group).
    1. In the storage account, select **Access control (IAM)**.
    1. Select **Add** and select the **Add role assignment** option.
    1. Search and select the **Storage Table Data Contributor**.
    1. Then select **Next**.
    1. In the **Members** tab, under **Assign access to**, select **Managed identity**.
    1. Select the **Azure Logic App** type.
    1. And then select the name of the managed identity associated with your logic app.
1. Next, create a Verified ID state table. This table will store the state of the presentation requests. 
    1. From the menu, select **Data Storage** and then select **Tables**.
    1. Select "+ table” and for the **Name**, enter `VerifiedIdState`.
    1. Select "Okay” to create the table.
1. Go the **Overview** page, and copy the storage account name.

## 5. Add the parameters for the workflows

In this step, add the required “parameters” for the workflows in the logic app. These parameters are values copied from the previous steps.

1. Open the Logic App you created.
1. From the menu, select **parameters**.
1. Copy the content of the [parameters.josn](./Azure-Logic-app/parameters.json) file and add it to the editor.
1. Edit the following:
    1. **TenantId** with your tenant ID that you copied earlier.
    1. **ClientId** of the application you registered.
    1. **ClientSecret** the application’s secret.
    1. **DidAuthority** the authority that uniquely identifies your verified id environment.
    1. **CallbackUrl** the callback URL which is the endpoint that Microsoft Entra Verified ID uses to notify your application about the request progress. You will update it later.
    1. **ApiKey** is the “API key” for securing callback notifications sent by Microsoft Entra verified ID to your application. You can generate a value like a secret.
    1. **StorageAccountName** with the storage account associated with your Azure Logic App.

## 6. Create the presentation workflow

With all components in place, except the mail service, proceed with building the workflows. When the helpdesk personnel request the user to verify their identity, the application interface calls this endpoint.

1. On the logic app menu, select **workflows**.
    1. Then select **add** and choose the **add** option.
    1. Enter a name for your workflow, like **Presentation**.
    1. You can choose either **Stateful** or **stateless**. For non-production environment, choose the **Stateful* option. It can help solve technical issues.
    1. Then select **create**.
    1. The new workflow will appear on the list, select it.

1. Next, edit the workflow. A workflow always begins with a “trigger”, which is an event or condition that specifies when the workflow should start, in this scenario it will start “on incoming HTTP request” and it’s followed by one or more actions, like sending a notification and returning a HTTP response.
    1. On the “designer surface”, select **add trigger**.
    1. Find the **Request** trigger and then select the **When a HTTP request is received**.
    1. Change the name of the action to `api`.The name of the “request trigger” determines the URL of your web API endpoint. The URL of the web API will be generated after you save the changes. 
    1. The custom authentication extension makes an HTTP POST request to your workflow. So, select the **POST** method. 
    1. For the **Request Body JSON Schema**, add the following:
    
    ```json
    {
      "type": "object",
      "properties": {
        "upn": {
          "type": "string"
        },
        "phone": {
          "type": "string"
        }
      }
    }
    ```
1.	The next step is to request an access token that can be used to call the Verified ID request service.
    1. Click on the **plus** button and select **add an action**.
    1. For the type select, **HTTP**.
    1. Change the name to `Obtain security token`.
    1.For the URL, enter: 
    
    ```
    https://login.microsoftonline.com/@{parameters('TenantId')}/oauth2/v2.0/token
    ```
    1.Change the HTTP **method** to `POST`.
    1. Add an **HTTP header** `Content-Type` with the value of `application/x-www-form-urlencoded`
    1. For the **body**, enter:
    
    ```
    grant_type=client_credentials&client_id=@{parameters('ClientId')}&client_secret=@{parameters('ClientSecret')}&scope=3db474b9-6a0c-4840-96ac-1fceb342124f/.default
    ```
        
1. The token endpoint provides a JSON document. In this step, you parse the JSON to retrieve the access token. 
    1. Add an action type of **Parse JSON**.
    1. Change the name of the action to `Parse token JSON`.
    1. For the **Content** enter: `@{body('Obtain_security_token')}`
    1. For the **Schema** enter:

    ```json
    {
      "type": "object",
      "properties": {
        "token_type": {
          "type": "string"
        },
        "expires_in": {
          "type": "integer"
        },
        "ext_expires_in": {
          "type": "integer"
        },
        "access_token": {
          "type": "string"
        }
      }
    }
    ```
     

1. With the access token obtained, you can now call the Microsoft Entra verified ID request endpoint.

    1. Add an action type of **HTTP**.
    1. Change the name to `Call Entra verified ID`.
    1. For the URL enter: 
    
    ```
    https://verifiedid.did.msidentity.com/v1.0/verifiableCredentials/createPresentationRequest
    ```
    
    1. Set the HTTP method to `POST`.
    1. Enter the following **body**:
    
    ```json
    {
      "authority": "@{parameters('DidAuthority')}",
      "includeQRCode": false,
      "registration": {
        "clientName": "Woodgrove helpdesk",
        "purpose": "Prove your identity"
      },
      "callback": {
        "url": "@{parameters('CallbackUrl')}",
        "state": "@{triggerBody()?['upn']}",
        "headers": {
          "api-key": "@{parameters('ApiKey')}"
        }
      },
      "includeReceipt": false,
      "requestedCredentials": [
        {
          "type": "VerifiedEmployee",
          "acceptedIssuers": [
            "@{parameters('DidAuthority')}"
          ],
          "configuration": {
            "validation": {
              "allowRevoked": false,
              "validateLinkedDomain": true
            }
          },
          "constraints": [
            {
              "claimName": "revocationId",
              "values": [
                "@{triggerBody()?['upn']}"
              ]
            }
          ]
        }
      ]
    }
    ```

    1. Under **advanced parameters** select **authentication**.
    1. For **authentication type**, select **raw**.
    1. And for the **value**, enter:
    
    ```
    Bearer @{body('Parse_token_JSON')?['access_token']}
    ```

1. Next, parse the response from the Microsoft Entra verified ID request endpoint.
    1. Add an action type of **Parse JSON**.
    1. Change the **name** to `Parse verified ID JSON response`.
    1. For the **Content**, add: `@{body('Call_Entra_verified_ID')}`
    1. For the **schema**, enter:
    
    ```json
    {
      "type": "object",
      "properties": {
        "requestId": {
          "type": "string"
        },
        "url": {
          "type": "string"
        },
        "expiry": {
          "type": "integer"
        }
      }
    }
    ``` 

1. Once the verified ID response is received, send the notification to the user and update the latest status in the state table. We'll add the email notification soon. First compose a JSON document.
    1. Add an action type of “Compose”
    1. Change the **name** to `Compose state JSON`.
    1. For the **Input**, add:
    
    ```json
    {
      "status": "request_created",
      "message": "Waiting for the user to open the notification.",
      "url": "@{body('Parse_verified_ID_JSON_response')?['url']}"
    }
    ```
    
1. To update the state table.	
    1. Add an action type of **Insert or Replace Entity (V2)**.
    1. For the first time you use a blob storage, you need to enter a connection:
        1. Enter a connection **Name** (the name is not important), like “ConnectionToStorageTable”
        1. Authentication Type, select: **Logic App Managed Identity**.
        1. Select “Create new” to create the connection.

    1. Continue with the action settings:
    1. Change the **Name** to `Update state table`.
    1. Partition **Key**, enter: `key`. Note, the name is not important.
    1. For the **Raw**, enter: `@{triggerBody()?['upn']}`
    1. For the **Storage account name or table endpoint**: enter: `@{parameters('StorageAccountName')}`
    1. For the **Table**, select `VerifiedIdState`.
    1. For the entity, enter: `@{outputs('Compose_state_JSON')}`. It's the JSON document you composed in the previous steps.
    1. 
1. The final step in the workflow is to return an HTTP response to the caller (your application).
    1. Add an action, type of **Response**.
    1. Make sure the status is `200`.
    1. For the Body, enter: `@{outputs('Compose_state_JSON')}`. It's the JSON document you composed in the previous steps.

## 7. Add the Callback workflows

The next workflow is the “callback endpoint”. Microsoft Entra verified ID invokes this endpoint to update the application with the request’s progress. 

1. On the logic app menu, select **workflows**.
1. Then select **add** and choose the **add** option.
1. Enter a name for your workflow, like **callback**.
1. You can choose either **Stateful** or **stateless**. For non-production environment, choose the **Stateful* option. It can help solve technical issues.
1. Then select **create**.
1. The new workflow will appear on the list, select it.
1. Edit the workflow. Switch to the **Code view** and paste the content of the [Callback/workflow.json](./Azure-Logic-app/Callback/workflow.json) and save the changes.

## 8. Add the Status workflows

The final workflow checks the state session status, reads the state cache, and returns it to your application.

1. On the logic app menu, select **Status**.
1. Then select **add** and choose the **add** option.
1. Enter a name for your workflow, like **Status**.
1. You can choose either **Stateful** or **stateless**. For non-production environment, choose the **Stateful* option. It can help solve technical issues.
1. Then select **create**.
1. The new workflow will appear on the list, select it, select it.
1. 1. Edit the workflow. Switch to the **Code view** and paste the content of the [Status/workflow.json](./Azure-Logic-app/Status/workflow.json) and save the changes.