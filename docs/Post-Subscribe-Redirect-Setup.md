# Post-Subscribe Redirect Configuration

## Overview

This guide explains how to configure the SaaS Accelerator to redirect customers to an external URL after they click the **Subscribe** button on the Customer Portal. By default, customers see an internal confirmation page. With this change, they are redirected to a URL of your choice (e.g., your solution's dashboard or onboarding page).

Once configured, the redirect URL is manageable from the **Admin Portal** (`/ApplicationConfig`) without redeployment.

---

## Prerequisites

- SaaS Accelerator deployed and running (Admin + Customer portals)
- Azure CLI (`az`) installed and logged in
- Access to the Azure SQL database used by the SaaS Accelerator
- Ability to rebuild and deploy the CustomerSite project

---

## Step 1: Add the Configuration Row to the Database

Connect to your SaaS Accelerator database and run the following SQL:

```sql
INSERT INTO ApplicationConfiguration ([Name], [Value], [Description])
VALUES (
  'PostSubscribeRedirectUrl',
  'https://your-target-url.com',
  'External URL to redirect the customer after clicking Subscribe. Leave empty to show the default confirmation page.'
);
```

Replace `https://your-target-url.com` with your desired redirect destination.

**Using Azure CLI with AAD auth:**

```powershell
sqlcmd -S <your-sql-server>.database.windows.net -d <your-database-name> `
  --authentication-method=ActiveDirectoryDefault `
  -Q "INSERT INTO ApplicationConfiguration ([Name], [Value], [Description]) VALUES ('PostSubscribeRedirectUrl', 'https://your-target-url.com', 'External URL to redirect the customer after clicking Subscribe. Leave empty to show the default confirmation page.');"
```

---

## Step 2: Modify the Controller (`HomeController.cs`)

**File:** `src/CustomerSite/Controllers/HomeController.cs`

**Method:** `SubscriptionOperationAsync`

Locate the line near the end of the method:

```csharp
this.notificationStatusHandlers.Process(subscriptionId);

return this.RedirectToAction(nameof(this.ProcessMessage), new { action = operation, status = operation });
```

Replace it with:

```csharp
this.notificationStatusHandlers.Process(subscriptionId);

// Check for external redirect URL after Subscribe
if (operation == "Activate")
{
    var postSubscribeRedirectUrl = this.applicationConfigRepository.GetValueByName("PostSubscribeRedirectUrl");
    if (!string.IsNullOrWhiteSpace(postSubscribeRedirectUrl))
    {
        if (Request.Headers["X-Requested-With"] == "XMLHttpRequest")
        {
            return Json(new { redirectUrl = postSubscribeRedirectUrl });
        }
        return this.Redirect(postSubscribeRedirectUrl);
    }
}

return this.RedirectToAction(nameof(this.ProcessMessage), new { action = operation, status = operation });
```

**What this does:**
- After the subscription is activated, it checks for a `PostSubscribeRedirectUrl` value in the database.
- If the request is an AJAX call (which the Subscribe button uses), it returns JSON with the redirect URL so the browser JavaScript can navigate.
- If it's a standard form POST, it does a server-side redirect.
- If the config value is empty or missing, the default confirmation page is shown (no change in behavior).

---

## Step 3: Modify the JavaScript (`_LandingPage.cshtml`)

**File:** `src/CustomerSite/Views/Home/_LandingPage.cshtml`

Locate the `SubscriptionOperation` JavaScript function and update the `$.ajax` call:

**Before:**

```javascript
$.ajax({
    url: '/Home/SubscriptionOperation',
    type: 'POST',
    headers: { RequestVerificationToken: csrftoken },
    data: formobject + "&subscriptionId=" + subscriptionId + "&planId=" + planId + "&operation=" + operation,
    cache: false,
    success: function (result) {
        $('#divIndex').html(result);
    },
    Error:
        function (result) {
            $('#divIndex').html(result);
        }
});
```

**After:**

```javascript
$.ajax({
    url: '/Home/SubscriptionOperation',
    type: 'POST',
    headers: { RequestVerificationToken: csrftoken, 'X-Requested-With': 'XMLHttpRequest' },
    data: formobject + "&subscriptionId=" + subscriptionId + "&planId=" + planId + "&operation=" + operation,
    cache: false,
    success: function (result) {
        if (result && result.redirectUrl) {
            window.location.href = result.redirectUrl;
        } else {
            $('#divIndex').html(result);
        }
    },
    Error:
        function (result) {
            $('#divIndex').html(result);
        }
});
```

**What this does:**
- Adds the `X-Requested-With: XMLHttpRequest` header so the controller can detect AJAX requests.
- In the success handler, checks if the response contains a `redirectUrl` and navigates the browser to it.
- Falls back to the default behavior (rendering HTML in the page) for all other operations.

---

## Step 4: Build and Deploy

```powershell
# Build
dotnet publish "src/CustomerSite/CustomerSite.csproj" -c Release -o "./publish/CustomerSite"

# Package
Compress-Archive -Path "./publish/CustomerSite/*" -DestinationPath "./publish/CustomerSite.zip" -Force

# Deploy to Azure App Service
az webapp deploy --resource-group <your-resource-group> --name <your-portal-app-name> --src-path "./publish/CustomerSite.zip" --type zip
```

---

## Step 5: Verify

1. Go to your Customer Portal (e.g., `https://<your-portal>.azurewebsites.net/Home/Subscriptions`).
2. Click **Subscribe** on a subscription that is pending activation.
3. The browser should redirect to your configured URL.

---

## Managing the Redirect URL

After initial setup, you can change the redirect URL at any time from the **Admin Portal**:

1. Go to `https://<your-admin-site>.azurewebsites.net/ApplicationConfig`
2. Find the **PostSubscribeRedirectUrl** setting
3. Update the value to a new URL, or clear it to restore the default confirmation page behavior

No redeployment is required to change the URL.

---

## Summary of Changes

| Component | File | Change |
|-----------|------|--------|
| Database | `ApplicationConfiguration` table | New row: `PostSubscribeRedirectUrl` |
| Controller | `src/CustomerSite/Controllers/HomeController.cs` | Return JSON redirect for AJAX, server redirect for standard POST |
| View | `src/CustomerSite/Views/Home/_LandingPage.cshtml` | Handle JSON redirect response in AJAX success callback |
