# Connecting to Google BigQuery with Azure Data Factory v2

---

> **TL;DR Version:**  You need to get a refresh token from Google.  Use
<a href="https://raw.githubusercontent.com/AnalyticJeremy/ADF_BigQuery/main/01%20Get%20Code%20URL.ps1">this PowerShell script</a>
to get an access code and
<a href="https://raw.githubusercontent.com/AnalyticJeremy/ADF_BigQuery/main/02%20Get%20Refresh%20Token.ps1">this PowerShell script</a>
to get a refresh token.

---

**Long Version**

Azure Data Factory includes a
[built-in connector to Google BigQuery](https://docs.microsoft.com/en-us/azure/data-factory/connector-google-bigquery).
However, the documentation is a bit sparse on how to authenticate between the two services.  This guide will provide some "lessons
learned" for configuring the connector.

## Authentication Types
<a href="media/new_linked_service.png" target="_blank"><img align="right" width="300" padding="20" src="media/new_linked_service.png"></a>
When configuring a new linked service for Google BigQuery, you must select the "Authentication type".  There are two choices:
User Authentication and Service Authentication.

**Service Authentication** allows you to provide a .p12 key file containing the credentials from Google.  However, this option is
only available when using the Self-Hosted Integration Runtime, which runs on an on-premises server.  If you want a cloud-to-cloud
only solution, you cannot use Service Authentication.

**User Authentication** uses OAuth 2 authorization to connect your Google user account to Azure Data Factory.  This option can be
used with a default Azure integration runtime for a cloud-only solution.  The remainder of this guide deals with this type of
authentication.

## Configuring "User Authentication"
When you configure user authentication for Azure Data Factory to work with Google BigQuery, the "New Linked Service" UI requires
that you provide three values when using "User Authentication".  These values are:
Client Id, Client Secret, and Refresh Token.  The first two values can be easily obtained from the GCP console.  The following
section shows how to get these two values.  Obtaining the "Refresh Token" is covered later in this guide.

### Creating New Client Id and Client Secret in GCP
You must use the [Google API Console](https://console.developers.google.com/) to obtain OAuth 2.0 credentials such as a
client ID and client secret that are known to both Google and Azure Data Factory.  Google publishes a document named
"[Using OAuth 2.0 for Web Server Applications](https://developers.google.com/identity/protocols/OAuth2WebServer)" that gives all
of the details for this process.  However, to make it easier, I will distill the steps here.

1. If you have not yet enabled Google APIs for your project, you must do that first. ([details](https://developers.google.com/identity/protocols/OAuth2WebServer#prerequisites))
2. In the Google Cloud Platform console, select your project.  Then select "Credentials" from the toolbar on the left side of the screen.
3. Click the "Create credentials" drop-down and select "OAuth client ID".
4. For "Application type", select "Web application".
   - Give the client a name that will allow you to identify it later.
   - The "Authorized JavaScript origins" field can be left blank.
   - For "Authorized redirect URIs", you must provide a URL (even if it's a fake URL).  Be sure to make note of the *exact* value that you use (including whether or not you used a trailing slash) because you will use it later.  I recommend you use:  `http://127.0.0.1/`
 5. Click the "Create" button.
 
<a href="media/create_cred_form.png" target="_blank"><img width="550" src="media/create_cred_form.png"></a>
 
The GCP Console will display the new client ID and client secret.  You can copy these values and paste them into the corresponding
fields in ADF's "New Linked Service" form.

<a href="media/oauth_creds.png" target="_blank"><img width="350" src="media/oauth_creds.png"></a>

### Obtaining a Refresh Token
The tricky part of this process is obtaining a refresh token.  To get the token,
you must call the Google authentication service REST API to get an access code.
Then you have to pass that access code back to Google to get a refresh token.

To make this process easier, I have two written PowerShell scripts (which are
available in this GitHub repo) you can use to do the "OAuth dance" with Google
and get a refresh token that you can provide to Azure Data Factory.

#### Step 1
First, you must run the
"<a href="https://raw.githubusercontent.com/AnalyticJeremy/ADF_BigQuery/main/01%20Get%20Code%20URL.ps1">01 Get Code URL</a>"
script. At the top of the script, just provide values for the `clientId`,
`clientSecret`, and `redirectUrl` variables.  We got all three of the values
in the previous section of this guide.  When you run the script, it will give
you a URL for `accounts.google.com`.  Open this URL in your web browser of choice.

You will be prompted to log in to your Google account.  You may receive a
warning that says "Google hasn't verified this app."  In this case, the "app"
is the application that you created above.  If you trust yourself, you can
click the "Advanced" link on the warning page and then click the (unsafe) link
at the bottom to go to your app.

<a href="media/oauth_window.png" target="_blank"><img width="300" src="media/oauth_window.png"></a>

Next, click the "Allow" button to grant your Google account access to the app
you created.  When you click this button, your web browser will be redirected
to the "redirect URL" that you provided when you created the app.  There will
be a long code added to the end of the URL that you provided.  Just copy the
whole URL from your browser's address bar.  The hard part of this process is
over, and you can proceed to step 2!

#### Step 2

Now you must run the
"<a href="https://raw.githubusercontent.com/AnalyticJeremy/ADF_BigQuery/main/02%20Get%20Refresh%20Token.ps1">02 Get Get Refresh Token</a>"
script.  At the top of the script, you must once again provide the same three
values you provied in step 1:  `clientId`, `clientSecret`, and `redirectUrl`.
There is a fourth variable you must also provide: `codeUrl`.  This is the
URL you copied from your browser at the end of Step 1.

Once you have plugged in the four values, run the PowerShell script.  It will
send the authorization code to Google and get a refresh token. The PowerShell
script will then show you the three values you need to configure the linked
service in Azure Data Factory, including the refresh token.

>**Note**: You can only use the authorization code URL one time.  If you try
to run Step 2 a second time, you will get a "bad request" error from Google.
If you need repeat this process, you must repeate Step 1 and then repeat Step 2.

Remember that the client secret and refresh token are keys that grant access to
your Google BigQuery data. Store them securely as you would any password. It is
a best practice to store these credentials in Azure Key Vault and link to them
in you Azure Data Factory linked service settings.

<a href="media/ps-result.png" target="_blank"><img width="600" src="media/ps-result.png"></a>

---
### See Also
My friend and colleague John Dandison has a write up that uses different tools
to accomplish this same goal.  If you would like a different perspective on
obtaining a Google refresh token, I highly recommend his post:
[Getting your BigQuery refresh_token for Azure DataFactory](https://jpda.dev/getting-your-bigquery-refresh-token-for-azure-datafactory-f884ff815a59)