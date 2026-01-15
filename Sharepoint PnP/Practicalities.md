How to get list of all list items for library, folders, files, etc 
```powershell
Measure-Command { 
  $all_list_items_for_library =  Get-PnPListItem -List $params.libraryName -PageSize 5000 -Fields "FileLeafRef","FileRef","File_x0020_Size","Created","Modified","Author","Editor"  
}
```
# SharePoint PnP Connection
## Connect with Certificate Authentication
```powershell
Connect-PnPOnline `
    -Url $params.siteUrl `
    -ClientId $params.clientId `
    -Tenant $params.tenant `
    -CertificatePath (Join-Path $PSScriptRoot "..\config\certs\SillenoProjectControl.pfx") `
    -CertificatePassword (ConvertTo-SecureString $params.clientSecret -AsPlainText -Force)
```
## Parameters
### `siteUrl`
SharePoint site URL.
**Example:** `https://exampledotorg.sharepoint.com/sites/TheSiteLikeProjectControl`
### `clientId`
Azure AD Application (Client) ID — a GUID from your app registration.
**How to obtain:**
- **Azure Portal:** App registrations → your app → Overview → Application (client) ID
- **PowerShell:**
  ```powershell
  Connect-AzAccount
  Get-AzADApplication | Select-Object DisplayName, AppId
  ```
**Example:** `a1b2c3d4-e5f6-7890-abcd-ef1234567890`
### `tenant`
Microsoft 365 tenant domain.
**Example:** `exampledotorg.onmicrosoft.com`