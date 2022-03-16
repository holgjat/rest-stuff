# Veeam Backup for Microsoft Azure comes with its own REST API

## Assumptions

- Veeam Backup for Microsoft Azure is deployed (you can find a free version on the Azure Marketplace)
- The Veeam appliance can be accessed through https
- Credentials are known

## Introduction

The Veeam REST API is very well documented in the [Veeam Backup for Microsoft Azure 3.0 REST API Reference](https://helpcenter.veeam.com/docs/vbazure/rest/introduction.html?ver=30).

Alternatively, you can explore the API through the [Swagger UI](https://helpcenter.veeam.com/docs/vbazure/rest/evaluation_swagger_ui.html?ver=30).

The PowerShell samples below are supposed to give an idea how to work with it through PowerShell.

How is this useful?! For automating deployment and management. If we want to automate the deployment and the initial setup of an environment, then this might come handy.

## Authenticate against the Veeam Server

The code below will get authenticate a user against the Veeam REST API and create the corresponding Bearer token. For the authentication, we need a Username and password (potentially an MFA pin - if configured) and the Veeam Server URI. That's it already. 

```azurepowershell
$Credential     = Get-Credential
$VeeamUri       = 'https://VEEAMSERVERURI'
$MfaCode        = 'MFACODE' #Optional

$requestMethod  = 'POST'
$requestBody    = '&Username='+$Credential.UserName+'&Password='+$($Credential.Password | ConvertFrom-SecureString -AsPlainText)+'&grant_type=Password'
$requestHeader  = @{ 
    'Content-Type' = 'application/x-www-form-urlencoded' 
}
$requestUri     = $VeeamUri+'/api/oauth2/token'

$request        = Invoke-RestMethod -Method $requestMethod `
                                    -Uri $requestUri `
                                    -Body $requestBody `
                                    -Headers $requestHeader `
                                    -SkipCertificateCheck
```

Note that the `-SkipCertificateCheck` shouldn't be used in a production environment where an appropriate certificate should be in place.

The `$request` variable is now holding the corresponding properties that we received from the REST API.

```azurepowershell
$request | Get-Member

   TypeName: System.Management.Automation.PSCustomObject

Name            MemberType   Definition
----            ----------   ----------
Equals          Method       bool Equals(System.Object obj)
GetHashCode     Method       int GetHashCode()
GetType         Method       type GetType()
ToString        Method       string ToString()
.expires        NoteProperty datetime .expires=16/03/2022 09:11:50
.issued         NoteProperty datetime .issued=16/03/2022 08:11:50
access_token    NoteProperty string access_token=eyJhbGciOi…
expires_in      NoteProperty long expires_in=3600
latestNewsShown NoteProperty bool latestNewsShown=True
mfa_enabled     NoteProperty bool mfa_enabled=False
refresh_token   NoteProperty string refresh_token=eyJhbGciOi…
roleName        NoteProperty string roleName=PortalAdmin
shortLived      NoteProperty bool shortLived=False
token_type      NoteProperty string token_type=bearer
userId          NoteProperty string userId=e727f243-c633-467d-a056-2e3a76683412
username        NoteProperty string username=<Veeam Username>
userType        NoteProperty string userType=Internal
```

The two properties that are of most interest are `token_type` and `access_token`. If we want to build the header for subsequent REST API requests, then we can use those properties to create the proper Bearer token.

```azurepowershell
$veeamToken = $($request.token_type+' '+$request.access_token)
```

If MFA is used, then a subsequent request is required in order for the final token to be returned.

```azurepowershell
$mfaRequest = Invoke-RestMethod -Method $requestMethod `
                                -Uri $requestUri `
                                -Body $('grant_type=Mfa&mfa_token='+$request.mfa_token+'&mfa_code='+$MfaCode) `
                                -Headers $requestHeader `
                                -SkipCertificateCheck
```

## How can we work with this from here?

### Just use the token in further requests

From here on, we can either use the above token to create requests to the REST API. For example for receiving all Backup Policies:

```azurepowershell
$requestMethod  = 'GET'
$requestUri     = $VeeamUri+'/api/v3/policies'
$requestHeader  = @{ 
                        'Content-Type' = 'application/json' 
                        Authorization  = $veeamToken
                    }

$request        = Invoke-RestMethod -Method $requestMethod `
                                    -Uri $requestUri `
                                    -Headers $requestHeader `
                                    -SkipCertificateCheck
```

### Create our own PowerShell functions

We could also just create our own PowerShell functions and store the token in a global variable.

```azurepowershell
function Connect-VeeamServer {
    [CmdletBinding()]
    param (
        [pscredential]$Credential,
        [string]$VeeamUri,
        [string]$MfaCode
    )
    $requestMethod  = 'POST'
    $requestBody    = '&Username='+$Credential.UserName+'&Password='+$($Credential.Password | ConvertFrom-SecureString -AsPlainText)+'&grant_type=Password'
    $requestHeader  = @{ 'Content-Type' = 'application/x-www-form-urlencoded' }
    $requestUri     = $VeeamUri+'/api/oauth2/token'

    $request        = Invoke-RestMethod -Method $requestMethod `
                                        -Uri $requestUri `
                                        -Body $requestBody `
                                        -Headers $requestHeader `
                                        -SkipCertificateCheck

    $Global:veeamConnectionPool = @{
        veeamServer = $VeeamUri
        veeamToken  = $null
        veeamUser   = $Credential.UserName
    }

    if($MfaCode) {
        $mfaRequest = Invoke-RestMethod -Method $requestMethod `
                                        -Uri $requestUri `
                                        -Body $('grant_type=Mfa&mfa_token='+$request.mfa_token+'&mfa_code='+$MfaCode) `
                                        -Headers $requestHeader `
                                        -SkipCertificateCheck

        $Global:veeamConnectionPool.veeamToken = $($mfaRequest.token_type+' '+$mfaRequest.access_token)
    }
    else {
        $Global:veeamConnectionPool.veeamToken = $($request.token_type+' '+$request.access_token)
    }
}
```

The global variable would then include the relevant information.

```azurepowershell
$veeamConnectionPool

Name                           Value
----                           -----
veeamToken                     bearer eyJhbGciOi…
veeamServer                    https://VEEAMSERVERURI
veeamUser                      <Veeam Username>
```

This is really just a sample. It lacks basic error checking and there are probably tons of other/better ways to do this but it's ok for playing around. Subsequent functions can then just make use of the information that is stored in the global variable without the need of passing token and server information over and over.

```azurepowershell
function Get-VeeamBackupPolicy {
    [CmdletBinding()]
    param ()

    if($Global:veeamConnectionPool.veeamToken) {
        $requestMethod  = 'GET'
        $requestUri     = $Global:veeamConnectionPool.veeamServer+'/api/v3/policies'
        $requestHeader  = @{ 
                                'Content-Type' = 'application/json' 
                                Authorization  = $Global:veeamConnectionPool.veeamToken
                            }

        $request        = Invoke-RestMethod -Method $requestMethod `
                                            -Uri $requestUri `
                                            -Headers $requestHeader `
                                            -SkipCertificateCheck

        return $request.results
    }
    else {
        Write-Output ("No token found. Please run 'Connect-VeeamServer'.")
    }
}
```

## References

1. [RFC 6750 ](https://datatracker.ietf.org/doc/html/rfc6750)
1. [Veeam Backup for Microsoft Azure 3.0 REST API Reference](https://helpcenter.veeam.com/docs/vbazure/rest/introduction.html?ver=30)
1. [Veeam Backup for Microsoft Azure 3.0 Release Notes](https://www.veeam.com/veeam_backup_microsoft_azure_3_0_release_notes_rn.pdf)
