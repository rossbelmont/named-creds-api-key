# How to Configure an API Key with Named Credentials in Salesforce
## Introduction
Though the HTTP standard includes an `Authorization` header, it’s relatively common for web services to define their own authentication protocols utilizing other headers. One simple and popular approach—often referred to as an “API key”—is to specify a header like `X-Client-ID` or `X-API-Key` and set its value to a shared secret that acts as a programmatic password.

Here’s a realistic example from [a service that provides mock data](https://mockaroo.com/) for testing:

```
GET https://api.mockaroo.com/api/11a5aea9?count=20 HTTP/1.1
X-API-Key: abc123
```

In this case, an HTTP request to the defined endpoint will return 20 sample records of a certain type to a calling application that identifies itself with the `X-API-Key` header. A developer would typically obtain this secret value from the configuration UI of the service itself. These values can sometimes be “reset” in case of abuse, though services like GitHub require developers to delete and recreate them as part of a reference to an external application allowed to access that particular user account.

The header itself can have an arbitrary name, including (but not limited to):
- `ACCESS_TOKEN`
- `Bearer`
- `Client-ID`
- `developer-token`
- `X-API-Key`

It’s also possible for *standard* headers like `Authorization` to be used along with the custom API keys. Here’s an example similar to the one above, but from [a service that provides stock photos](https://unsplash.com/):

```
GET https://api.unsplash.com/photos/random HTTP/1.1
Authorization: Client-ID abc123
```

This approach is more closely aligned with the HTTP spec covering [Basic authentication](https://en.wikipedia.org/wiki/Basic_access_authentication), but still deviates from the standard in important ways.

The larger point is that the web service owner defines these rules, the headers, and the acceptable values. It is, in effect, a custom authentication protocol and not a standard like OAuth or JWT. Named Credentials in Salesforce now include a **Custom** authentication protocol to support this popular pattern.

## Configuration Walkthrough
To configure a Named Credential with API key like the above example, we’ll touch on each element in the following order:
- The container for the custom authentication scheme, stored as the External Credential
- The secret value, stored as an Authorization Parameter and protected by a Permission Set
- The Custom Header, which captures the header itself and references the secret value via a merge field
- The endpoint URL, stored with the Named Credential

All four are needed. To begin, navigate to **Setup** → **Named Credentials** to access the correct area of Setup. 

### External Credential: a Container for Authentication Configuration
The **External Credential** is a container that encapsulates all authentication details needed to access a particular endpoint. It does *not* contain the endpoint URL itself; that is captured on the Named Credential object (discussed later). The two are separate so that multiple endpoints can be addressed with the same authentication. This would apply when, say, building integrations against both the Google Calendar API and the Google Drive API using Google’s OAuth-based authentication provider. (Users would only need to sign in once.)

For Custom authentication protocols, there are only a few configuration values to specify on the External Credential object.

- Click the **External Credentials** subtab, then click **New**.
- Enter a descriptive **Label**.
- Enter an unambiguous developer **Name**.
- Set the **Authentication Protocol** to Custom.
- Click **Save**.

![Creating the External Credential container](https://github.com/rossbelmont/named-creds-api-key/blob/main/screenshots/Ext%20Cred%20for%20API%20Key%202022-08-27.jpeg?raw=true)

We’ll use the developer Name in one of the steps below, and you’ll see the Label in the very last step.

### Storing the API Key as an Authentication Parameter in a Permission Set Mapping
To make Named Credentials secure by default in every possible sense, we want to carefully consider how and where we store the secret API key value. Salesforce strongly recommends capturing such values as Authentication Parameters as part of a **Permission Set Mapping**. These parameters are expected to contain secrets, and as such, are stored in an encrypted manner consistent with other sensitive information on our platform. 

Additionally, access to the secrets is granted explicitly by way of a Permission Set. As the name suggests, a Permission Set Mapping maps a Permission Set to one or more Authentication Parameters. This way, different groups of Salesforce users can use different authentication tokens to call the same external service. 

In the case of the stock photo service here, perhaps some users could download a commercially-licensed photo (incurring a cost), while others can only choose from the no-cost options. Two different Permission Sets would be mapped to two different sets of Authentication Parameters, each containing a different API key.

The stock photo service refers to the API key as a `Client-ID`, so we create a Permission Set Mapping as follows:

- When viewing the External Credential, click **New** under **Permission Set Mappings**.
- Select a **Permission Set** that users will need to access this secret value. (Note: you may want to create a new Permission Set for this purpose.)
- Set the **Sequence Number** in case a user happens to have multiple Permission Sets used in multiple Permission Set Mappings. This number determines which mapping “wins” and gets used for the callout, sorted from lowest to highest.
- Leave the **Identity Type** set to Named Principal. This indicates that Salesforce users will share the same API key, and they don’t have unique access to the external service.
    - *Note: at this time, only the OAuth protocol supports unique Per User access to a remote system. In that case, each user logs in separately before the integration works in their user context.*
- Click **Add** to create an Authentication Parameter, and use `MyClientId` as the **Name**. The Name is a developer name that will be used in a later step.
- Paste in the API key as the **Value** of the Authentication Parameter.
- Click **Save**.

![Storing the API key as an Authentication Parameter](https://github.com/rossbelmont/named-creds-api-key/blob/main/screenshots/Storing%20API%20Key%20as%20Auth%20Param%202022-08-27.jpeg?raw=true)

There’s one other aspect of encrypted storage administrators need to understand. The encrypted secrets are stored in a Standard Object called **User External Credentials**, and any user that will make callouts needs Create, Read, Update, and Delete access to this object. The record manipulation is managed by the platform—users can’t navigate a record detail page for a User External Credential and view raw tokens—but object access still needs to be explicitly granted. 

Technically, this is independent of any individual Named Credential. After you’ve created all the relevant Named Credentials, make sure to open up access to this object via Permission Sets or Profiles before your non-administrative users need to test/use this functionality.

![Allow access to User External Credential object](https://github.com/rossbelmont/named-creds-api-key/blob/main/screenshots/Grant%20UCE%20Access%202022-08-23.gif?raw=true)

Using a data object for this token storage allows higher volumes of tokens to be stored by the platform, which is critical when using Per User access with the OAuth protocol and each user logs into the external system’s UI separately. Each login yields a separate token, yielding tens of thousands of tokens in a large enterprise.

### Custom Headers with Merge Fields and Formula Support
**Custom Headers** allow external web services to define arbitrary parameters needed as input to respond to a certain request. It's similar to a function in a piece of code having parameters or arguments that allow the caller to supply input.

Since these headers are often used for authentication, Salesforce allows administrators to define them as part of both Named Credentials and External Credentials. This example defines the `Authorization` header on the External Credential.

The value of the header illustrates the use of both merge fields and formulas, which can be combined in powerful ways. **Merge fields** provide access to encrypted values via the `$Credential.Container.ParameterName` syntax, and in this case the container is the External Credential. **Formulas** can be used in header values via the `{!FormulaGoesHere}` syntax; anything inside `{!}` will be evaluated as a [Salesforce formula](https://help.salesforce.com/s/articleView?id=customize_functions.htm&type=5&language=en_US). Formulas provide significant power and flexibility to craft header values without coding.

Our stock photo example combines both:

- When viewing the External Credential, click **New** under **Custom Headers**.
- Enter the **Name** of the header as required by the external service (`Authorization`, in this case). 
- Enter the following **Value**:`{!'Client-ID ' & $Credential.UnsplashCustomAuthScheme.MyClientId}`
- Leave the **Sequence Number** set to the default, since we don’t need to worry about the order of headers in this case.
- Click **Save**.

![Defining the Custom Header](https://github.com/rossbelmont/named-creds-api-key/blob/main/screenshots/Custom%20Header%20for%20API%20Key%202022-08-27.jpeg?raw=true)

Looking at the Value more closely, we see that the literal string `Client-ID` (followed by a space) is concatenated with the API key:

```
{!'Client-ID ' & $Credential.UnsplashCustomAuthScheme.MyClientId}
```

If the `Client-ID` were `abc123`, this would be the resulting callout:

```
GET https://api.unsplash.com/photos/random HTTP/1.1
Authorization: Client-ID abc123
```

Here's a view of how all the External Credential configuration comes together.

![The overall External Credential](https://github.com/rossbelmont/named-creds-api-key/blob/main/screenshots/Overall%20Ext%20Cred%20Config%20for%20API%20Key%202022-08-27.jpeg?raw=true)

#### Basic Authentication
One of the many options provided by formulas is the capability to build an `Authorization` header that works with HTTP's [Basic authentication protocol](https://en.wikipedia.org/wiki/Basic_access_authentication). That protocol uses Base64 encoding, which can be accomplished with the help of the `BASE64` function.

Step through the same fundamental process above:

- Create an External Credential with a developer Name of `BasicAuth`. (Refer to the first section above to understand the steps.)
- Define two Authentication Parameters in the Permission Set Mapping, for example `UsernameParam` and `PasswordParam`. (Refer to the second section above to understand the steps.)
- Use the following steps to define the `Authorization` header with a different formula. 

With the External Credential in place, and the Authentication Parameters defined, you're ready to add a Custom Header with a formula matching what's required for the Basic authentication protocol.

- When viewing the new External Credential, click **New** under **Custom Headers**.
- Enter the **Name** of the header as required by the external service (`Authorization`, in this case). 
- Enter the following **Value**:`{!'Basic ' & BASE64($Credential.BasicAuth.UsernameParam & ":" & $Credential.BasicAuth.PasswordParam)}`
- Leave the **Sequence Number** set to the default, or set the order explicitly if you're concerned about collisions.
- Click **Save**.

#### API Keys without Formulas
If a service has slightly simpler authentication, and the entire value of the header is the API key itself, then formulas are not needed (since there’s no concatenation or other text processing). The **Name** would be something like `X-API-Key` and the **Value** would be `$Credential.MyExtCred.APIKey`. Note that this requires an Authentication Parameter named `APIKey` is defined as part of a Permission Set Mapping to store the secret value. (See above.)

### Storing the Endpoint URL in the Named Credential
Now that the trickier work of defining the External Credential is complete, an administrator can define the  **Named Credential**  that references it. The Named Credential stores the endpoint URL, as well as a few other parameters that may need to be adjusted for a particular endpoint. 

- Click the **Named Credentials** subtab, then click **New**.
- Enter a descriptive **Label**.
- Enter an unambiguous developer **Name**.
- Set the **URL** to the correct endpoint, which in this case references our stock photo service.
- Select the  **External Credential** created in the prior step.
- For this example, it’s important to *uncheck* the **Generate Authorization Header** setting. Instead, this will be handled by the Custom Header we defined in the prior steps.
- Similar to the above, it’s important to *check* the **Allow Formulas in HTTP Header** setting so that the formula gets evaluated.
- Click **Save**.

![Defining the Named Credential](https://github.com/rossbelmont/named-creds-api-key/blob/main/screenshots/Named%20Cred%20for%20API%20Key%202022-08-27.jpeg?raw=true)

At this point, a callout using our Named Credential will return successfully, since it has the correct `Authorization` header. If the tokens expire or the URL changes, no changes to Apex code will be needed. In addition to Apex,the credential can be used in no-code tools like [External Services](https://help.salesforce.com/s/articleView?id=sf.external_services.htm&type=5) that provide [integration with Flow](https://help.salesforce.com/s/articleView?id=sf.enhanced_external_services_example_create_flow.htm&type=5).
