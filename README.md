# Helpdesk initiated prove request

The diagram below outlines the components and the flow of a presentation request:

![Presentation request diagram](./help/flow.png)

## Verifier endpoint

The verifier endpoint initiates a presentation request when helpdesk staff ask an employee to verify their identity.

- **Step 1.** To initiate the verification process, a helpdesk personnel enters the user's UPN and phone number (to which the notification will be sent). A JavaScript code then calls the [verifier endpoint](./Azure-Logic-app/Presentation/workflow.json). Use the following JSON payload to call the verifier endpoint from the frontend app:
    
    ```json
    {
        "upn": "10@sample.com",
        "phone": ""
    }
    ```

- **Step 2.** Preparation
  - **Step 2.1** Conducts several input validations, such as verifying the validity of the UPN.
  
  - **Step 2.2** Acquires an access token utilizing the client credentials flow through an application registration in Microsoft Entra ID (with the respective permissions).

  - **Step 2.3**  Generates a JSON payload like the one below. The payload includes the `request.Callback.State` with a generated unique identifier. This ID is crucial for correlating the callback event with the state included in the original payload and it's used as the key for the Azure Blob table object. 

- **Step 3.** Call to the **Request Service REST API**. 
  - **Step 3.1** With both the JSON payload and access token ready, it subsequently starts an HTTP POST request to the Microsoft Entra verified ID *Request Service REST API*. Here is an example of the JSON payload:

    ```json
    {
        "authority": "did:web:did.woodgrove.com",
        "includeQRCode": false,
        "registration": {
            "clientName": "Woodgrove helpdesk",
            "purpose": "Prove your identity"
        },
        "callback": {
            "url": "https://woodgrove.com/api/verifier/presentationcallback",
            "state": "11111111-0000-0000-0000-000000000000",
            "headers": {
                "api-key": "<API key to help protect your callback API>"
            }
        },
        "includeReceipt": false,
        "requestedCredentials": [
            {
                "type": "VerifiedEmployee",
                "acceptedIssuers": [
                    "did:web:did.woodgrove.com"
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
                            "10@sample.com"
                        ]
                    }
                ]
            }
        ]
    }
    ```

  - **Step 3.2** If the **Request Service REST API** call is successful, it returns a response code (HTTP 201 Created), and a collection of event objects in the response body.The `url` is that launches the authenticator app and starts the presentation process. Applications can present this URL through a web hyperlink, QR code, or push notification. The following JSON demonstrates a successful response:

    ```json
    {
        "requestId": "e4ef27ca-eb8c-4b63-823b-3b95140eac11",
        "url": "openid-vc://?request_uri=https://verifiedid.did.msidentity.com/v1.0/00001111-aaaa-2222-bbbb-3333cccc4444/verifiableCredentials/presentationRequests/e4ef27ca-eb8c-4b63-823b-3b95140eac11",
        "expiry": 1633017751
    }
    ```
    
 - **Step 4.** The **verifier endpoint** uses Azure Blob table to keep state about the request. The key is the state value from the `request.Callback.State` attribute in the request. The UI will check the state when calling the [status endpoint](./Controllers/RequestStatusController.cs). Finally, the **verifier endpoint** returns the status to the frontend application. Then the frontend application uses this data to update the user interface. The following JSON shows an example.
    
    ```json
    {
        ...
        "RowKey": "yoel@woodgrovecorp.onmicrosoft.com",
        "Timestamp": "2025-04-20T07:39:39.8996443Z",
        "message": "Waiting for the user to open the notification.",
        "status": "request_created",
        "url": "openid-vc://?request_uri=https://verifiedid.did.msidentity.com/v1.0/00001111-aaaa-2222-bbbb-3333cccc4444/verifiableCredentials/presentationRequests/e4ef27ca-eb8c-4b63-823b-3b95140eac11"
    }
    ```

## Callback endpoint

- **Step 5.** The [callback endpoint](./Azure-Logic-app/Callback/workflow.json) is called (by Microsoft Entra verified ID service) when a user scans the QR code, uses the deep link to the Authenticator app, or finishes the presentation process. The payload contains information like the `state` value that you passed in the original payload (we use it as a key for Azure Blob table), `requestStatus`, `claims` and more. 


- **Step 6.** The callback endpoint executes the logic:

  - **Step 6.1** Verifies if the `api-key` HTTP header matches the `ApiKey` parameter; if not, returns a 401 unauthorized error.
  - **Step 6.2** Checks the value of the `requestStatus` and creates a user friendly status message.
  - **Step 6.3** Updates the Azure Blob table with the latest status and the friendly status message, so the UI can check the state when calling the [status endpoint](./Controllers/RequestStatusController.cs).
    

The following JSON shows an example of **request_retrieved** status to the callback endpoint

```json
{
    "requestId": "12345678-1234-1234-1234-000000000000",
    "requestStatus": "request_retrieved",
    "state": "11111111-0000-0000-0000-000000000000"
}
```


The following JSON shows an example of **presentation_verified** status to the callback endpoint
```json
{
    "requestId": "12345678-1234-1234-1234-000000000000",
    "requestStatus": "presentation_verified",
    "state": "11111111-0000-0000-0000-000000000000",
    "verifiedCredentialsData": [
        {
            "issuer": "did:web:did.woodgrove.com",
            "type": [
                "VerifiableCredential",
                "VerifiedEmployee"
            ],
            "claims": {
                "displayName": "Test user",
                "preferredLanguage": "en-US",
                "revocationId": "10@sample.com",
                "photo": "iVBORw0KGBCZr8J8yiLFAtu4JHpJ...."
            },
            "credentialState": {
                "revocationStatus": "VALID"
            },
            "domainValidation": {
                "url": "https://did.woodgrove.com/"
            },
            "expirationDate": "2025-09-15T11:29:12.000Z",
            "issuanceDate": "2025-03-19T11:29:12.000Z"
        }
    ],
    "subject": "did:ion:EiBxCxgghGgBvNEVi9E...."
}
```
    
## Status endpoint

The frontend application queries the [status endpoint](./Azure-Logic-app/Status/workflow.json) every three seconds to obtain the latest updates regarding user activity using the user's principal name. The endpoint retrieves the status data from the Azure Blob table and return it to the application. Consequently, the application updates the user interface in accordance with this information. When the status is `presentation_verified`, the application should stop the three-second interval.

The JSON below illustrates the payload required to call the status endpoint:

```json
{
    "state": "10@sample.com"
}
```

The response is similar to the one returned verifier endpoint

```json
{
    ...
    "RowKey": "yoel@woodgrovecorp.onmicrosoft.com",
    "Timestamp": "2025-04-20T07:41:30.8996443Z",
    "message": "The user has successfully completed their verification process.",
    "status": "presentation_verified"
}
```

## Architecture

The diagram below illustrates the components of the solution and their interactions:

![Architecture design diagram](./help/architecture.png)

## Azure Logic App

Azure Logic Apps provides a low-code/no-code platform for building workflows and automating processes. The HTTP Request trigger allows you to create a Logic App that can be triggered by an HTTP request. This effectively turns the Logic App into a web API endpoints, including:

- [Verifier](./Azure-Logic-app/Presentation/workflow.json) (knowns as presentation endpoint)
- [Status](./Azure-Logic-app/Status/workflow.json) 
- [Callback](./Azure-Logic-app/Callback/workflow.json)  

### Push notifications

In this sample, the Logic App’s verifier endpoint retrieves the `phone` from the HTTP request payload and uses Azure Email Communication Service to send the URL of the presentation request. The transmit method can be altered to phone SMS or WhatsApp messages using Azure Communication services. 

### Parameters

The [parameters](./Azure-Logic-app/parameters.json) necessary for the workflows

- **ApiKey** a secret used by the API
- **CallbackUrl** - This is your Logic App callback workflow. The URL must be simple for Entra ID to call it. We use Azure API management service with URL [rewrite to remove extra query string parameters](./Azure-API-Management/APIM-policy.xml) and simplify the URL.
- **TenantId** - Your Microsoft Entra tenant ID.
- **ClientId** - The application ID you registered for the verified ID.
- **ClientSecret** - The application secret you created.
- **Scope** - Use this `3db474b9-6a0c-4840-96ac-1fceb342124f/.default` value.
- **DidAuthority** - The DID identifier of your verified ID (from [Entra admin center](https://entra.microsoft.com))

### API Connections

- The `azuretables` [API Connections](./Azure-Logic-app/connections.json) enable the reading and writing of entities within the Azure Blob table.
- The `acsemail` [API Connections](./Azure-Logic-app/connections.json) is used to send email via Azure Email Communication Service.

## Azure Table Storage

Azure Table Storage is a service that stores structured NoSQL data in the cloud. We use this service to persists the presentation flow state for the users, including:

- **PartitionKey** set to 'State'
- **RowKey** is the user state identifier (user UPN).
- Request status
- User-friendly message
- The URI to initiate the presentation flow upon user interaction

## Azure API Management

Azure API Management functions as a gateway for all requests from frontend application and Microsoft Entra. The requests first reach the APIM gateway, which then forwards them to respective backend services (using URL rewrite).

1. It simplifies the complex URL of Logic Apps, making it consumable by Entra ID.
1. Handles Cross-Origin Resource Sharing (CORS) for Logic App endpoints. This ensures secure access from different origins, which is important for single-page applications and AJAX calls that need to make cross-domain requests.
1. [Not implemented] Use OAuth 2.0 authorization with Microsoft Entra ID to secure access to the **verifier** and **status** endpoints. The frontend application must pass a Bearer token to the web API. Additionally, access to the endpoint should be restricted through Azure APIM to prevent direct access. Note that the callback endpoint should NOT be secured with an OAuth 2.0 authorization Bearer token. Instead, the callback endpoint verifies the API key. 

## Frontend application

The frontend application renders the UI elements that users interact with. It uses jQuery to call the **verifier** and **status** endpoints.

1. The Logic App endpoints (via Azure APIM) should set CORS as mentioned above.
1. [Not implemented] Use the MSAL library or Azure Static Apps Easy Auth to obtain an access token for calling the Logic Apps endpoints.
1. [Not implemented] input validation.

