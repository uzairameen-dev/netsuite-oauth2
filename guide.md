# Connect an external app to a NetSuite RESTlet

Machine-to-machine access to a NetSuite RESTlet with **SuiteTalk OAuth 2.0 client credentials**:
the caller signs a PS256 JWT with its private key, exchanges it for an access token,
caches the token, and calls the RESTlet with `Authorization: Bearer <token>`.

This file is the agent-readable version of the interactive guide. Follow it as a spec.

## Reference

| Item | Value |
|---|---|
| Account ID in hostnames | Lowercase, `_` replaced with `-` (`1234567_SB1` becomes `1234567-sb1`) |
| Token endpoint | `POST https://<account>.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token` |
| RESTlet endpoint | `https://<account>.restlets.api.netsuite.com/app/site/hosting/restlet.nl?script=<scriptId>&deploy=<deployId>` |
| JWT header | `{"alg": "PS256", "typ": "JWT", "kid": "<certificate ID from the M2M mapping>"}` |
| JWT payload | `{"iss": "<client ID>", "scope": ["restlets"], "aud": "<token endpoint URL>", "iat": <now>, "exp": <now + 3600 at most>}` |
| Token request body | `application/x-www-form-urlencoded`: `grant_type=client_credentials`, `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`, `client_assertion=<signed JWT>` |
| Token response | `{"access_token": "...", "expires_in": "3600", "token_type": "Bearer"}`. `expires_in` may be a string or a number |
| Token lifetime | About 60 minutes |
| RESTlet request | `Authorization: Bearer <access_token>`, `Content-Type: application/json`, JSON body |
| Key types | RSA 3072 or 4096-bit, signed with PSS (PS256). SuiteScript signing needs an RSA-PSS key |
| Certificate validity | 2 years maximum |

Rules every client must follow:

1. Cache the access token in memory and reuse it until 60 seconds before it expires.
2. Let concurrent callers share one in-flight token request (lock, mutex, or shared promise).
3. Parse `expires_in` whether it is a string or a number.
4. Never log the token request: it contains the signed assertion.
5. Keep the private key out of source control and container images. Load it from a secret store or a file path in an environment variable.

## Environment variables

All examples read these names:

```bash
NS_ACCOUNT_ID=1234567-sb1                    # lowercase, "_" -> "-"
NS_CLIENT_ID=paste-your-client-id            # integration record (step 3)
NS_CERTIFICATE_ID=paste-your-certificate-id  # M2M certificate mapping (step 5)
NS_PRIVATE_KEY_PATH=./private.pem
NS_RESTLET_SCRIPT_ID=customscript_order_sync # script ID or internal ID
NS_RESTLET_DEPLOY_ID=customdeploy_order_sync # deployment ID or internal ID
```

## Setup in NetSuite

1. **Enable features.** Setup > Company > Enable Features > SuiteCloud: check *Server SuiteScript*, *REST Web Services*, and *OAuth 2.0* (under Manage Authentication).
2. **Create the integration role.** Setup > Users/Roles > Manage Roles > New. Permissions: *Log in using OAuth 2.0 Access Tokens* (Full), *REST Web Services* (Full), plus only the record permissions the RESTlet uses. Assign the role to a dedicated integration user (employee record > Access > Roles).
3. **Create the integration record.** Setup > Integration > Manage Integrations > New. Check *Client Credentials (Machine to Machine) Grant*, scope *RESTlets* (add *REST Web Services* if calling the record API). Uncheck TBA and Authorization Code Grant if unused. Copy the **Client ID**; it is shown once.
4. **Generate a key pair.** See [Key generation](#key-generation).
5. **Map the certificate.** Setup > Integration > OAuth 2.0 Client Credentials (M2M) Setup > Create New. Entity = integration user, Role = integration role, Application = integration record, Certificate = `public.pem`. Copy the **Certificate ID** (used as the JWT `kid`).
6. **Deploy the RESTlet.** Deployment *Status* = Released (Testing only works for the owner). *Audience* must include the integration role. Note the script and deployment IDs.
7. **Fill in `.env`** with the values above.
8. **Call the RESTlet** with one of the [client examples](#client-examples).

## Key generation

> **Recommendation:** generate your own certificate with OpenSSL on the machine or secret store
> where the key will live. Use certificates from the interactive guide's in-browser generator
> only for testing, and replace them with your own before going live.

OpenSSL (recommended, any machine):

```bash
openssl req -new -x509 -newkey rsa:3072 -nodes -days 365 -sha256 \
  -keyout private.pem -out public.pem -subj "/CN=restlet-integration"
```

In the browser with Web Crypto and `@peculiar/x509`, for testing only (the key never leaves the device):

```javascript
// npm install @peculiar/x509   (or load build/x509.js, which defines a global x509)
import * as x509 from "@peculiar/x509";

x509.cryptoProvider.set(crypto); // the browser's Web Crypto

export async function generateKeyPair({ commonName = "restlet-integration", bits = 3072, days = 365 } = {}) {
  const alg = {
    name: "RSA-PSS",
    hash: "SHA-256",
    publicExponent: new Uint8Array([1, 0, 1]),
    modulusLength: bits,
    saltLength: 32,
  };
  const keys = await crypto.subtle.generateKey(alg, true, ["sign", "verify"]);

  // Random positive serial number
  const serial = crypto.getRandomValues(new Uint8Array(16));
  serial[0] &= 0x7f;

  const notBefore = new Date();
  const cert = await x509.X509CertificateGenerator.createSelfSigned({
    serialNumber: Array.from(serial, (b) => b.toString(16).padStart(2, "0")).join(""),
    name: `CN=${commonName}`,
    notBefore,
    notAfter: new Date(notBefore.getTime() + days * 24 * 60 * 60 * 1000),
    signingAlgorithm: alg,
    keys,
  });

  const publicPem = cert.toString("pem");
  const privatePem = x509.PemConverter.encode(
    await crypto.subtle.exportKey("pkcs8", keys.privateKey),
    "PRIVATE KEY"
  );
  return { publicPem, privatePem };
}

// Save a file from the browser
export function download(filename, text) {
  const url = URL.createObjectURL(new Blob([text], { type: "application/x-pem-file" }));
  const link = Object.assign(document.createElement("a"), { href: url, download: filename });
  link.click();
  setTimeout(() => URL.revokeObjectURL(url), 1000);
}

// Usage
const { publicPem, privatePem } = await generateKeyPair({ bits: 3072, days: 365 });
download("public.pem", publicPem);
download("private.pem", privatePem);
```

Browser-generated keys use the standard RSA key type. They work with every external client and with the NetSuite certificate mapping. For the *SuiteScript (other account)* case, generate an RSA-PSS key with OpenSSL instead (command in that example's setup).

## Sample RESTlet (SuiteScript 2.1)

```javascript
/**
 * @NApiVersion 2.1
 * @NScriptType Restlet
 */
define(["N/record", "N/log"], (record, log) => {
  const post = (body) => {
    log.audit("Order sync request", JSON.stringify(body));

    const so = record.create({ type: record.Type.SALES_ORDER, isDynamic: true });
    so.setValue({ fieldId: "externalid", value: body.externalId });
    // ...set entity, lines and other fields from body
    const id = so.save();

    return { success: true, id };
  };

  const get = (params) => {
    const so = record.load({ type: record.Type.SALES_ORDER, id: params.id });
    return { id: params.id, status: so.getText({ fieldId: "status" }) };
  };

  return { post, get };
});
```

## Client examples

Each example signs the JWT, caches the access token, and POSTs JSON to the RESTlet.

### Node.js

Node 18+ (built-in fetch). Reads settings from `.env`.

Install / setup:

```bash
npm install jsonwebtoken dotenv
```

`restlet-client.js`:

```javascript
require("dotenv").config();
const fs = require("fs");
const jwt = require("jsonwebtoken");

const {
  NS_ACCOUNT_ID, NS_CLIENT_ID, NS_CERTIFICATE_ID,
  NS_PRIVATE_KEY_PATH, NS_RESTLET_SCRIPT_ID, NS_RESTLET_DEPLOY_ID,
} = process.env;

const TOKEN_URL = `https://${NS_ACCOUNT_ID}.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token`;
const RESTLET_URL = `https://${NS_ACCOUNT_ID}.restlets.api.netsuite.com/app/site/hosting/restlet.nl`;

let cachedToken = null;
let tokenExpiresAt = 0;
let pendingToken = null;

// Returns the cached token until a minute before it expires.
async function getAccessToken() {
  if (cachedToken && Date.now() < tokenExpiresAt - 60_000) return cachedToken;
  // Concurrent callers share one in-flight token request
  pendingToken ??= requestAccessToken().finally(() => { pendingToken = null; });
  return pendingToken;
}

async function requestAccessToken() {
  const now = Math.floor(Date.now() / 1000);
  const assertion = jwt.sign(
    { iss: NS_CLIENT_ID, scope: ["restlets"], aud: TOKEN_URL, iat: now, exp: now + 3600 },
    fs.readFileSync(NS_PRIVATE_KEY_PATH),
    { algorithm: "PS256", header: { kid: NS_CERTIFICATE_ID, typ: "JWT" } }
  );

  const res = await fetch(TOKEN_URL, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "client_credentials",
      client_assertion_type: "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
      client_assertion: assertion,
    }),
  });
  if (!res.ok) throw new Error(`Token request failed: ${res.status} ${await res.text()}`);
  const data = await res.json();
  cachedToken = data.access_token;
  tokenExpiresAt = Date.now() + Number(data.expires_in) * 1000;
  return cachedToken;
}

async function postToRestlet(body) {
  const token = await getAccessToken();
  const url = new URL(RESTLET_URL);
  url.searchParams.set("script", NS_RESTLET_SCRIPT_ID);
  url.searchParams.set("deploy", NS_RESTLET_DEPLOY_ID);

  const res = await fetch(url, {
    method: "POST",
    headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  if (!res.ok) throw new Error(`RESTlet call failed: ${res.status} ${await res.text()}`);
  return res.json();
}

postToRestlet({ externalId: "ORD-1001", lines: [{ sku: "ABC-1", qty: 2 }] })
  .then((result) => console.log(result))
  .catch((err) => console.error(err.message));
```

### Python

PyJWT signs PS256 through the `cryptography` package.

Install / setup:

```bash
pip install pyjwt cryptography requests python-dotenv
```

`restlet_client.py`:

```python
import os
import threading
import time

import jwt
import requests
from dotenv import load_dotenv

load_dotenv()

ACCOUNT = os.environ["NS_ACCOUNT_ID"]
CLIENT_ID = os.environ["NS_CLIENT_ID"]
CERT_ID = os.environ["NS_CERTIFICATE_ID"]
KEY_PATH = os.environ["NS_PRIVATE_KEY_PATH"]
SCRIPT_ID = os.environ["NS_RESTLET_SCRIPT_ID"]
DEPLOY_ID = os.environ["NS_RESTLET_DEPLOY_ID"]

TOKEN_URL = f"https://{ACCOUNT}.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token"
RESTLET_URL = f"https://{ACCOUNT}.restlets.api.netsuite.com/app/site/hosting/restlet.nl"


_token = None
_token_expires_at = 0.0
_token_lock = threading.Lock()


def get_access_token() -> str:
    """Return the cached token, requesting a new one a minute before it expires."""
    global _token, _token_expires_at
    with _token_lock:
        if _token is None or time.time() >= _token_expires_at - 60:
            data = _request_access_token()
            _token = data["access_token"]
            _token_expires_at = time.time() + int(data["expires_in"])
        return _token


def _request_access_token() -> dict:
    now = int(time.time())
    with open(KEY_PATH, "rb") as f:
        private_key = f.read()

    assertion = jwt.encode(
        {"iss": CLIENT_ID, "scope": ["restlets"], "aud": TOKEN_URL, "iat": now, "exp": now + 3600},
        private_key,
        algorithm="PS256",
        headers={"kid": CERT_ID, "typ": "JWT"},
    )

    res = requests.post(
        TOKEN_URL,
        data={
            "grant_type": "client_credentials",
            "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
            "client_assertion": assertion,
        },
        timeout=30,
    )
    if not res.ok:
        raise RuntimeError(f"Token request failed: {res.status_code} {res.text}")
    return res.json()


def post_to_restlet(body: dict) -> dict:
    res = requests.post(
        RESTLET_URL,
        params={"script": SCRIPT_ID, "deploy": DEPLOY_ID},
        json=body,
        headers={"Authorization": f"Bearer {get_access_token()}"},
        timeout=60,
    )
    if not res.ok:
        raise RuntimeError(f"RESTlet call failed: {res.status_code} {res.text}")
    return res.json()


if __name__ == "__main__":
    print(post_to_restlet({"externalId": "ORD-1001", "lines": [{"sku": "ABC-1", "qty": 2}]}))
```

### Java

Java 17+. Nimbus JOSE + JWT for PS256, built-in HttpClient. Export `.env` values as environment variables.

Install / setup:

```xml
<!-- pom.xml (Java 17+) -->
<dependency>
  <groupId>com.nimbusds</groupId>
  <artifactId>nimbus-jose-jwt</artifactId>
  <version>9.37.3</version>
</dependency>

# Java has no built-in .env loader. Export the variables first:
#   set -a; source .env; set +a
```

`NetSuiteRestletClient.java`:

```java
import com.nimbusds.jose.JOSEObjectType;
import com.nimbusds.jose.JWSAlgorithm;
import com.nimbusds.jose.JWSHeader;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jose.util.JSONObjectUtils;
import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.SignedJWT;

import java.net.URI;
import java.net.URLEncoder;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.Base64;
import java.util.Date;
import java.util.List;
import java.util.Map;

public class NetSuiteRestletClient {
    static final String ACCOUNT = System.getenv("NS_ACCOUNT_ID");
    static final String CLIENT_ID = System.getenv("NS_CLIENT_ID");
    static final String CERT_ID = System.getenv("NS_CERTIFICATE_ID");
    static final String KEY_PATH = System.getenv("NS_PRIVATE_KEY_PATH");
    static final String SCRIPT_ID = System.getenv("NS_RESTLET_SCRIPT_ID");
    static final String DEPLOY_ID = System.getenv("NS_RESTLET_DEPLOY_ID");

    static final String TOKEN_URL = "https://" + ACCOUNT + ".suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token";
    static final String RESTLET_URL = "https://" + ACCOUNT + ".restlets.api.netsuite.com/app/site/hosting/restlet.nl";

    static final HttpClient HTTP = HttpClient.newHttpClient();

    // Expects a PKCS#8 key ("BEGIN PRIVATE KEY"), which OpenSSL 3 produces by default.
    static PrivateKey loadPrivateKey(String path) throws Exception {
        String pem = Files.readString(Path.of(path))
                .replaceAll("-----(BEGIN|END) PRIVATE KEY-----", "")
                .replaceAll("\\s", "");
        byte[] der = Base64.getDecoder().decode(pem);
        return KeyFactory.getInstance("RSA").generatePrivate(new PKCS8EncodedKeySpec(der));
    }

    static String cachedToken;
    static long tokenExpiresAt;

    // Returns the cached token until a minute before it expires.
    static synchronized String getAccessToken() throws Exception {
        if (cachedToken == null || System.currentTimeMillis() >= tokenExpiresAt - 60_000) {
            Map<String, Object> data = requestAccessToken();
            cachedToken = (String) data.get("access_token");
            // NetSuite may send expires_in as a string or a number
            long expiresIn = Long.parseLong(String.valueOf(data.get("expires_in")));
            tokenExpiresAt = System.currentTimeMillis() + expiresIn * 1000;
        }
        return cachedToken;
    }

    static Map<String, Object> requestAccessToken() throws Exception {
        Date now = new Date();
        JWTClaimsSet claims = new JWTClaimsSet.Builder()
                .issuer(CLIENT_ID)
                .audience(TOKEN_URL)
                .claim("scope", List.of("restlets"))
                .issueTime(now)
                .expirationTime(new Date(now.getTime() + 3_600_000))
                .build();
        JWSHeader header = new JWSHeader.Builder(JWSAlgorithm.PS256)
                .keyID(CERT_ID)
                .type(JOSEObjectType.JWT)
                .build();
        SignedJWT jwt = new SignedJWT(header, claims);
        jwt.sign(new RSASSASigner(loadPrivateKey(KEY_PATH)));

        String form = "grant_type=client_credentials"
                + "&client_assertion_type=" + enc("urn:ietf:params:oauth:client-assertion-type:jwt-bearer")
                + "&client_assertion=" + enc(jwt.serialize());

        HttpResponse<String> res = HTTP.send(
                HttpRequest.newBuilder(URI.create(TOKEN_URL))
                        .header("Content-Type", "application/x-www-form-urlencoded")
                        .POST(HttpRequest.BodyPublishers.ofString(form))
                        .build(),
                HttpResponse.BodyHandlers.ofString());
        if (res.statusCode() != 200) {
            throw new IllegalStateException("Token request failed: " + res.statusCode() + " " + res.body());
        }
        return JSONObjectUtils.parse(res.body());
    }

    static String postToRestlet(String jsonBody) throws Exception {
        String url = RESTLET_URL + "?script=" + enc(SCRIPT_ID) + "&deploy=" + enc(DEPLOY_ID);
        HttpResponse<String> res = HTTP.send(
                HttpRequest.newBuilder(URI.create(url))
                        .header("Authorization", "Bearer " + getAccessToken())
                        .header("Content-Type", "application/json")
                        .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
                        .build(),
                HttpResponse.BodyHandlers.ofString());
        if (res.statusCode() >= 300) {
            throw new IllegalStateException("RESTlet call failed: " + res.statusCode() + " " + res.body());
        }
        return res.body();
    }

    static String enc(String value) {
        return URLEncoder.encode(value, StandardCharsets.UTF_8);
    }

    public static void main(String[] args) throws Exception {
        String body = """
                {"externalId": "ORD-1001", "lines": [{"sku": "ABC-1", "qty": 2}]}
                """;
        System.out.println(postToRestlet(body));
    }
}
```

### C#

.NET 8, no NuGet packages. `RSASignaturePadding.Pss` gives PS256.

Install / setup:

```bash
# .NET 8 console app, no extra packages needed
dotnet new console -n NetSuiteRestletClient
# Replace Program.cs with the code below, then export the .env values and run:
set -a; source .env; set +a
dotnet run
```

`Program.cs`:

```csharp
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

string Env(string name) =>
    Environment.GetEnvironmentVariable(name) ?? throw new InvalidOperationException($"Missing {name}");

var account = Env("NS_ACCOUNT_ID");
var clientId = Env("NS_CLIENT_ID");
var certId = Env("NS_CERTIFICATE_ID");
var keyPath = Env("NS_PRIVATE_KEY_PATH");
var scriptId = Env("NS_RESTLET_SCRIPT_ID");
var deployId = Env("NS_RESTLET_DEPLOY_ID");

var tokenUrl = $"https://{account}.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token";
var restletUrl = $"https://{account}.restlets.api.netsuite.com/app/site/hosting/restlet.nl";

using var http = new HttpClient();

string? cachedToken = null;
var tokenExpiresAt = DateTimeOffset.MinValue;
var tokenLock = new SemaphoreSlim(1, 1);

var result = await PostToRestletAsync(new
{
    externalId = "ORD-1001",
    lines = new[] { new { sku = "ABC-1", qty = 2 } },
});
Console.WriteLine(result);

// Returns the cached token until a minute before it expires.
async Task<string> GetAccessTokenAsync()
{
    await tokenLock.WaitAsync();
    try
    {
        if (cachedToken is null || DateTimeOffset.UtcNow >= tokenExpiresAt.AddSeconds(-60))
        {
            var (token, expiresIn) = await RequestAccessTokenAsync();
            cachedToken = token;
            tokenExpiresAt = DateTimeOffset.UtcNow.AddSeconds(expiresIn);
        }
        return cachedToken!;
    }
    finally
    {
        tokenLock.Release();
    }
}

async Task<(string Token, int ExpiresIn)> RequestAccessTokenAsync()
{
    var now = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
    var header = new { alg = "PS256", typ = "JWT", kid = certId };
    var payload = new { iss = clientId, scope = new[] { "restlets" }, aud = tokenUrl, iat = now, exp = now + 3600 };
    var signingInput = $"{B64Url(JsonSerializer.SerializeToUtf8Bytes(header))}.{B64Url(JsonSerializer.SerializeToUtf8Bytes(payload))}";

    using var rsa = RSA.Create();
    rsa.ImportFromPem(File.ReadAllText(keyPath));
    var signature = rsa.SignData(Encoding.ASCII.GetBytes(signingInput), HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
    var assertion = $"{signingInput}.{B64Url(signature)}";

    var res = await http.PostAsync(tokenUrl, new FormUrlEncodedContent(new Dictionary<string, string>
    {
        ["grant_type"] = "client_credentials",
        ["client_assertion_type"] = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
        ["client_assertion"] = assertion,
    }));
    var body = await res.Content.ReadAsStringAsync();
    if (!res.IsSuccessStatusCode) throw new Exception($"Token request failed: {(int)res.StatusCode} {body}");
    var root = JsonDocument.Parse(body).RootElement;
    // NetSuite may send expires_in as a string or a number
    var expires = root.GetProperty("expires_in");
    var expiresIn = expires.ValueKind == JsonValueKind.String ? int.Parse(expires.GetString()!) : expires.GetInt32();
    return (root.GetProperty("access_token").GetString()!, expiresIn);
}

async Task<string> PostToRestletAsync(object payload)
{
    var url = $"{restletUrl}?script={Uri.EscapeDataString(scriptId)}&deploy={Uri.EscapeDataString(deployId)}";
    using var req = new HttpRequestMessage(HttpMethod.Post, url)
    {
        Content = new StringContent(JsonSerializer.Serialize(payload), Encoding.UTF8, "application/json"),
    };
    req.Headers.Authorization = new("Bearer", await GetAccessTokenAsync());

    var res = await http.SendAsync(req);
    var body = await res.Content.ReadAsStringAsync();
    if (!res.IsSuccessStatusCode) throw new Exception($"RESTlet call failed: {(int)res.StatusCode} {body}");
    return body;
}

static string B64Url(byte[] bytes) =>
    Convert.ToBase64String(bytes).TrimEnd('=').Replace('+', '-').Replace('/', '_');
```

### PHP

PHP's OpenSSL extension can't make PSS signatures, so phpseclib signs. Token is cached in a static variable and in APCu when available.

Install / setup:

```bash
composer require phpseclib/phpseclib:~3.0 vlucas/phpdotenv
```

`restlet_client.php`:

```php
<?php
require __DIR__ . '/vendor/autoload.php';

use phpseclib3\Crypt\PublicKeyLoader;
use phpseclib3\Crypt\RSA;

Dotenv\Dotenv::createImmutable(__DIR__)->load();

$account = $_ENV['NS_ACCOUNT_ID'];
$tokenUrl = "https://{$account}.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token";
$restletUrl = "https://{$account}.restlets.api.netsuite.com/app/site/hosting/restlet.nl";

function b64url(string $data): string
{
    return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
}

function httpPost(string $url, string $body, array $headers): array
{
    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => $body,
        CURLOPT_HTTPHEADER => $headers,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 60,
    ]);
    $response = curl_exec($ch);
    if ($response === false) {
        throw new RuntimeException(curl_error($ch));
    }
    $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    return [$status, $response];
}

// Returns the cached token until a minute before it expires. PHP-FPM starts each request
// fresh, so the token is also kept in APCu (when installed) to share it across requests.
function getAccessToken(string $tokenUrl): string
{
    static $token = null;
    static $expiresAt = 0;
    if ($token !== null && time() < $expiresAt - 60) {
        return $token;
    }

    $useApcu = function_exists('apcu_enabled') && apcu_enabled();
    if ($useApcu) {
        $cached = apcu_fetch('netsuite_access_token');
        if (is_array($cached) && time() < $cached['expiresAt'] - 60) {
            [$token, $expiresAt] = [$cached['token'], $cached['expiresAt']];
            return $token;
        }
    }

    $data = requestAccessToken($tokenUrl);
    $token = $data['access_token'];
    $expiresAt = time() + (int) $data['expires_in'];
    if ($useApcu) {
        apcu_store('netsuite_access_token', ['token' => $token, 'expiresAt' => $expiresAt], (int) $data['expires_in']);
    }
    return $token;
}

function requestAccessToken(string $tokenUrl): array
{
    $now = time();
    $header = b64url(json_encode(['alg' => 'PS256', 'typ' => 'JWT', 'kid' => $_ENV['NS_CERTIFICATE_ID']]));
    $payload = b64url(json_encode([
        'iss' => $_ENV['NS_CLIENT_ID'],
        'scope' => ['restlets'],
        'aud' => $tokenUrl,
        'iat' => $now,
        'exp' => $now + 3600,
    ], JSON_UNESCAPED_SLASHES));

    $key = PublicKeyLoader::load(file_get_contents($_ENV['NS_PRIVATE_KEY_PATH']))
        ->withPadding(RSA::SIGNATURE_PSS)
        ->withHash('sha256')
        ->withMGFHash('sha256')
        ->withSaltLength(32);
    $assertion = "{$header}.{$payload}." . b64url($key->sign("{$header}.{$payload}"));

    [$status, $body] = httpPost($tokenUrl, http_build_query([
        'grant_type' => 'client_credentials',
        'client_assertion_type' => 'urn:ietf:params:oauth:client-assertion-type:jwt-bearer',
        'client_assertion' => $assertion,
    ]), ['Content-Type: application/x-www-form-urlencoded']);

    if ($status !== 200) {
        throw new RuntimeException("Token request failed: {$status} {$body}");
    }
    return json_decode($body, true);
}

function postToRestlet(string $restletUrl, string $tokenUrl, array $data): array
{
    $url = $restletUrl . '?' . http_build_query([
        'script' => $_ENV['NS_RESTLET_SCRIPT_ID'],
        'deploy' => $_ENV['NS_RESTLET_DEPLOY_ID'],
    ]);
    [$status, $body] = httpPost($url, json_encode($data), [
        'Authorization: Bearer ' . getAccessToken($tokenUrl),
        'Content-Type: application/json',
    ]);
    if ($status >= 300) {
        throw new RuntimeException("RESTlet call failed: {$status} {$body}");
    }
    return json_decode($body, true);
}

print_r(postToRestlet($restletUrl, $tokenUrl, [
    'externalId' => 'ORD-1001',
    'lines' => [['sku' => 'ABC-1', 'qty' => 2]],
]));
```

### Go

golang-jwt's PS256 uses a 32-byte salt, as NetSuite expects.

Install / setup:

```bash
go get github.com/golang-jwt/jwt/v5 github.com/joho/godotenv
```

`main.go`:

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"sync"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/joho/godotenv"
)

func mustEnv(name string) string {
	v := os.Getenv(name)
	if v == "" {
		log.Fatalf("missing environment variable %s", name)
	}
	return v
}

var (
	tokenMu        sync.Mutex
	cachedToken    string
	tokenExpiresAt time.Time
)

// getAccessToken returns the cached token until a minute before it expires.
func getAccessToken(tokenURL string) (string, error) {
	tokenMu.Lock()
	defer tokenMu.Unlock()
	if cachedToken != "" && time.Now().Before(tokenExpiresAt.Add(-time.Minute)) {
		return cachedToken, nil
	}

	token, expiresIn, err := requestAccessToken(tokenURL)
	if err != nil {
		return "", err
	}
	cachedToken = token
	tokenExpiresAt = time.Now().Add(time.Duration(expiresIn) * time.Second)
	return cachedToken, nil
}

func requestAccessToken(tokenURL string) (string, int, error) {
	keyPEM, err := os.ReadFile(mustEnv("NS_PRIVATE_KEY_PATH"))
	if err != nil {
		return "", 0, err
	}
	key, err := jwt.ParseRSAPrivateKeyFromPEM(keyPEM)
	if err != nil {
		return "", 0, err
	}

	now := time.Now()
	token := jwt.NewWithClaims(jwt.SigningMethodPS256, jwt.MapClaims{
		"iss":   mustEnv("NS_CLIENT_ID"),
		"scope": []string{"restlets"},
		"aud":   tokenURL,
		"iat":   now.Unix(),
		"exp":   now.Add(time.Hour).Unix(),
	})
	token.Header["kid"] = mustEnv("NS_CERTIFICATE_ID")
	assertion, err := token.SignedString(key)
	if err != nil {
		return "", 0, err
	}

	res, err := http.PostForm(tokenURL, url.Values{
		"grant_type":            {"client_credentials"},
		"client_assertion_type": {"urn:ietf:params:oauth:client-assertion-type:jwt-bearer"},
		"client_assertion":      {assertion},
	})
	if err != nil {
		return "", 0, err
	}
	defer res.Body.Close()
	body, _ := io.ReadAll(res.Body)
	if res.StatusCode != http.StatusOK {
		return "", 0, fmt.Errorf("token request failed: %d %s", res.StatusCode, body)
	}

	var tok struct {
		AccessToken string      `json:"access_token"`
		ExpiresIn   json.Number `json:"expires_in"` // NetSuite may send a string or a number
	}
	if err := json.Unmarshal(body, &tok); err != nil {
		return "", 0, err
	}
	expiresIn, err := strconv.Atoi(tok.ExpiresIn.String())
	if err != nil {
		return "", 0, err
	}
	return tok.AccessToken, expiresIn, nil
}

func postToRestlet(restletURL, tokenURL string, payload any) ([]byte, error) {
	accessToken, err := getAccessToken(tokenURL)
	if err != nil {
		return nil, err
	}
	data, err := json.Marshal(payload)
	if err != nil {
		return nil, err
	}

	q := url.Values{"script": {mustEnv("NS_RESTLET_SCRIPT_ID")}, "deploy": {mustEnv("NS_RESTLET_DEPLOY_ID")}}
	req, err := http.NewRequest(http.MethodPost, restletURL+"?"+q.Encode(), bytes.NewReader(data))
	if err != nil {
		return nil, err
	}
	req.Header.Set("Authorization", "Bearer "+accessToken)
	req.Header.Set("Content-Type", "application/json")

	res, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer res.Body.Close()
	body, _ := io.ReadAll(res.Body)
	if res.StatusCode >= 300 {
		return nil, fmt.Errorf("RESTlet call failed: %d %s", res.StatusCode, body)
	}
	return body, nil
}

func main() {
	_ = godotenv.Load()
	account := mustEnv("NS_ACCOUNT_ID")
	tokenURL := "https://" + account + ".suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token"
	restletURL := "https://" + account + ".restlets.api.netsuite.com/app/site/hosting/restlet.nl"

	body, err := postToRestlet(restletURL, tokenURL, map[string]any{
		"externalId": "ORD-1001",
		"lines":      []map[string]any{{"sku": "ABC-1", "qty": 2}},
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(string(body))
}
```

### Ruby

ruby-jwt supports PS256 with the openssl gem 2.1+.

Install / setup:

```bash
gem install jwt dotenv
```

`restlet_client.rb`:

```ruby
require "dotenv/load"
require "json"
require "jwt"
require "net/http"
require "openssl"

ACCOUNT = ENV.fetch("NS_ACCOUNT_ID")
TOKEN_URL = "https://#{ACCOUNT}.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token"
RESTLET_URL = "https://#{ACCOUNT}.restlets.api.netsuite.com/app/site/hosting/restlet.nl"

TOKEN_MUTEX = Mutex.new
@token = nil
@token_expires_at = 0

# Returns the cached token until a minute before it expires.
def access_token
  TOKEN_MUTEX.synchronize do
    if @token.nil? || Time.now.to_i >= @token_expires_at - 60
      data = request_access_token
      @token = data.fetch("access_token")
      # NetSuite may send expires_in as a string or a number
      @token_expires_at = Time.now.to_i + Integer(data.fetch("expires_in"))
    end
    @token
  end
end

def request_access_token
  key = OpenSSL::PKey::RSA.new(File.read(ENV.fetch("NS_PRIVATE_KEY_PATH")))
  now = Time.now.to_i
  assertion = JWT.encode(
    { iss: ENV.fetch("NS_CLIENT_ID"), scope: ["restlets"], aud: TOKEN_URL, iat: now, exp: now + 3600 },
    key,
    "PS256",
    { kid: ENV.fetch("NS_CERTIFICATE_ID"), typ: "JWT" }
  )

  res = Net::HTTP.post_form(
    URI(TOKEN_URL),
    grant_type: "client_credentials",
    client_assertion_type: "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    client_assertion: assertion
  )
  raise "Token request failed: #{res.code} #{res.body}" unless res.is_a?(Net::HTTPSuccess)

  JSON.parse(res.body)
end

def post_to_restlet(body)
  uri = URI(RESTLET_URL)
  uri.query = URI.encode_www_form(
    script: ENV.fetch("NS_RESTLET_SCRIPT_ID"),
    deploy: ENV.fetch("NS_RESTLET_DEPLOY_ID")
  )

  res = Net::HTTP.post(
    uri,
    body.to_json,
    "Authorization" => "Bearer #{access_token}",
    "Content-Type" => "application/json"
  )
  raise "RESTlet call failed: #{res.code} #{res.body}" unless res.is_a?(Net::HTTPSuccess)

  JSON.parse(res.body)
end

puts post_to_restlet(externalId: "ORD-1001", lines: [{ sku: "ABC-1", qty: 2 }])
```

### Dart

Server or CLI only. Never ship the private key inside a Flutter or other client app.

Install / setup:

```bash
dart pub add dart_jsonwebtoken http dotenv
```

`bin/restlet_client.dart`:

```dart
import 'dart:convert';
import 'dart:io';

import 'package:dart_jsonwebtoken/dart_jsonwebtoken.dart';
import 'package:dotenv/dotenv.dart';
import 'package:http/http.dart' as http;

final env = DotEnv(includePlatformEnvironment: true)..load();

String envVar(String name) =>
    env[name] ?? (throw StateError('Missing environment variable $name'));

final account = envVar('NS_ACCOUNT_ID');
final tokenUrl = 'https://$account.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token';
final restletUrl = 'https://$account.restlets.api.netsuite.com/app/site/hosting/restlet.nl';

String? _cachedToken;
DateTime _tokenExpiresAt = DateTime.fromMillisecondsSinceEpoch(0);
Future<String>? _pendingToken;

/// Returns the cached token until a minute before it expires.
Future<String> getAccessToken() {
  final token = _cachedToken;
  if (token != null &&
      DateTime.now().isBefore(_tokenExpiresAt.subtract(const Duration(minutes: 1)))) {
    return Future.value(token);
  }
  // Concurrent callers share one in-flight token request
  return _pendingToken ??= _requestAccessToken().whenComplete(() => _pendingToken = null);
}

Future<String> _requestAccessToken() async {
  final privateKey = RSAPrivateKey(File(envVar('NS_PRIVATE_KEY_PATH')).readAsStringSync());
  final jwt = JWT(
    {'iss': envVar('NS_CLIENT_ID'), 'scope': ['restlets'], 'aud': tokenUrl},
    header: {'kid': envVar('NS_CERTIFICATE_ID'), 'typ': 'JWT'},
  );
  // sign() adds iat and exp
  final assertion = jwt.sign(
    privateKey,
    algorithm: JWTAlgorithm.PS256,
    expiresIn: const Duration(hours: 1),
  );

  final res = await http.post(Uri.parse(tokenUrl), body: {
    'grant_type': 'client_credentials',
    'client_assertion_type': 'urn:ietf:params:oauth:client-assertion-type:jwt-bearer',
    'client_assertion': assertion,
  });
  if (res.statusCode != 200) {
    throw HttpException('Token request failed: ${res.statusCode} ${res.body}');
  }

  final data = jsonDecode(res.body) as Map<String, dynamic>;
  _cachedToken = data['access_token'] as String;
  // NetSuite may send expires_in as a string or a number
  final expiresIn = int.parse('${data['expires_in']}');
  _tokenExpiresAt = DateTime.now().add(Duration(seconds: expiresIn));
  return _cachedToken!;
}

Future<dynamic> postToRestlet(Map<String, dynamic> body) async {
  final uri = Uri.parse(restletUrl).replace(queryParameters: {
    'script': envVar('NS_RESTLET_SCRIPT_ID'),
    'deploy': envVar('NS_RESTLET_DEPLOY_ID'),
  });
  final res = await http.post(
    uri,
    headers: {
      'Authorization': 'Bearer ${await getAccessToken()}',
      'Content-Type': 'application/json',
    },
    body: jsonEncode(body),
  );
  if (res.statusCode >= 300) {
    throw HttpException('RESTlet call failed: ${res.statusCode} ${res.body}');
  }
  return jsonDecode(res.body);
}

Future<void> main() async {
  final result = await postToRestlet({
    'externalId': 'ORD-1001',
    'lines': [
      {'sku': 'ABC-1', 'qty': 2},
    ],
  });
  print(result);
}
```

### SuiteScript (same account)

`N/https.requestRestlet` authenticates as the running script's user and role. No OAuth, keys or token cache.

Install / setup:

```bash
# No packages to install. In NetSuite:
# 1. Upload the file to the File Cabinet, e.g. SuiteScripts/call_order_sync.js
# 2. Customization > Scripting > Scripts > New, pick the file, save
# 3. Create a deployment for it and run it (Save and Execute)
#
# The role running this script must be in the RESTlet deployment's Audience.
```

`SuiteScripts/call_order_sync.js`:

```javascript
/**
 * Calls a RESTlet in the same NetSuite account from server-side SuiteScript.
 * N/https.requestRestlet signs the request as the user and role running this
 * script, so no OAuth token, key pair or token cache is needed.
 *
 * @NApiVersion 2.1
 * @NScriptType ScheduledScript
 */
define(["N/https", "N/log"], (https, log) => {
  const SCRIPT_ID = "customscript_order_sync";
  const DEPLOY_ID = "customdeploy_order_sync";

  const postToRestlet = (body) => {
    const res = https.requestRestlet({
      scriptId: SCRIPT_ID,
      deploymentId: DEPLOY_ID,
      method: https.Method.POST,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });

    if (res.code >= 300) {
      throw new Error(`RESTlet call failed: ${res.code} ${res.body}`);
    }
    return JSON.parse(res.body);
  };

  const execute = () => {
    const result = postToRestlet({ externalId: "ORD-1001", lines: [{ sku: "ABC-1", qty: 2 }] });
    log.audit("RESTlet response", JSON.stringify(result));
  };

  return { execute };
});
```

### SuiteScript (other account)

Signs the JWT with `N/crypto/certificate` using an RSA-PSS certificate stored in the calling account. Token cached in `N/cache` (TTL minimum 300 s).

Install / setup:

```bash
# TARGET account (the one with the RESTlet):
#   Do steps 1-3, 5 and 6 of this guide. In step 5, upload public-pss.pem
#   from below and copy the Certificate ID.

# SOURCE account (the one running this script):
# 1. Generate an RSA-PSS key. SuiteScript signs PS256 only with an RSA-PSS key,
#    so the plain RSA key from step 4 won't work here.
openssl req -new -x509 -nodes -days 365 -newkey rsa-pss -pkeyopt rsa_keygen_bits:4096 \
  -subj "/CN=ns-target-integration" -out public-pss.pem -keyout private-pss.pem
cat public-pss.pem private-pss.pem > combined-pss.pem

# 2. Setup > Company > Certificates: upload combined-pss.pem with the
#    ID "_ns_target_pss" (NetSuite saves it as custcertificate_ns_target_pss).
#
# 3. Upload this script and create the script record. Add Free-Form Text parameters:
#      custscript_target_account     target account ID, e.g. 7654321-sb1
#      custscript_target_client_id   client ID of the integration in the target account
#      custscript_target_cert_id     Certificate ID from the target account's M2M setup
#      custscript_target_script_id   e.g. customscript_order_sync
#      custscript_target_deploy_id   e.g. customdeploy_order_sync
#
# 4. Create a deployment, fill in the parameters, and run it.
```

`SuiteScripts/call_remote_restlet.js`:

```javascript
/**
 * Calls a RESTlet in ANOTHER NetSuite account using OAuth 2.0 client credentials.
 * The JWT is signed with an RSA-PSS certificate stored in this account
 * (Setup > Company > Certificates), and the access token is cached with N/cache.
 *
 * @NApiVersion 2.1
 * @NScriptType ScheduledScript
 */
define(["N/https", "N/encode", "N/crypto/certificate", "N/cache", "N/runtime", "N/log"], (
  https,
  encode,
  certificate,
  cache,
  runtime,
  log
) => {
  // Certificate in THIS account that holds the private key
  const SIGNING_CERT_ID = "custcertificate_ns_target_pss";
  const TOKEN_CACHE_NAME = "ns_target_oauth";

  const getConfig = () => {
    const script = runtime.getCurrentScript();
    const param = (name) => {
      const value = script.getParameter({ name });
      if (!value) throw new Error(`Missing script parameter ${name}`);
      return value;
    };
    // Target account ID as used in URLs: lowercase, "_" replaced with "-"
    const account = param("custscript_target_account");
    return {
      clientId: param("custscript_target_client_id"),
      certId: param("custscript_target_cert_id"), // Certificate ID from the target account's M2M setup
      scriptId: param("custscript_target_script_id"),
      deployId: param("custscript_target_deploy_id"),
      tokenUrl: `https://${account}.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token`,
      restletUrl: `https://${account}.restlets.api.netsuite.com/app/site/hosting/restlet.nl`,
    };
  };

  const base64Url = (text) =>
    encode
      .convert({
        string: text,
        inputEncoding: encode.Encoding.UTF_8,
        outputEncoding: encode.Encoding.BASE_64_URL_SAFE,
      })
      .replace(/=+$/, "");

  const createAssertion = (config) => {
    const now = Math.floor(Date.now() / 1000);
    const header = base64Url(JSON.stringify({ alg: "PS256", typ: "JWT", kid: config.certId }));
    const payload = base64Url(
      JSON.stringify({ iss: config.clientId, scope: ["restlets"], aud: config.tokenUrl, iat: now, exp: now + 300 })
    );

    const signer = certificate.createSigner({
      certId: SIGNING_CERT_ID,
      algorithm: certificate.HashAlg.SHA256, // SHA-256 with an RSA-PSS key = PS256
    });
    signer.update({ input: `${header}.${payload}` });
    const signature = signer.sign({ outputEncoding: encode.Encoding.BASE_64_URL_SAFE }).replace(/=+$/, "");

    return `${header}.${payload}.${signature}`;
  };

  // Returns the cached token until a minute before it expires. Script executions
  // don't share memory, so the token lives in N/cache. N/cache can evict entries
  // early; a new token is then requested.
  const getAccessToken = (config) => {
    const tokenCache = cache.getCache({ name: TOKEN_CACHE_NAME, scope: cache.Scope.PRIVATE });
    const cached = tokenCache.get({ key: config.clientId });
    if (cached) return cached;

    const res = https.request({
      method: https.Method.POST,
      url: config.tokenUrl,
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: [
        "grant_type=client_credentials",
        `client_assertion_type=${encodeURIComponent("urn:ietf:params:oauth:client-assertion-type:jwt-bearer")}`,
        `client_assertion=${encodeURIComponent(createAssertion(config))}`,
      ].join("&"),
    });
    if (res.code !== 200) {
      throw new Error(`Token request failed: ${res.code} ${res.body}`);
    }

    const data = JSON.parse(res.body);
    // N/cache requires a TTL of at least 300 seconds
    const ttl = Math.max(300, Number(data.expires_in) - 60);
    tokenCache.put({ key: config.clientId, value: data.access_token, ttl });
    return data.access_token;
  };

  const postToRestlet = (config, body) => {
    const url =
      `${config.restletUrl}?script=${encodeURIComponent(config.scriptId)}` +
      `&deploy=${encodeURIComponent(config.deployId)}`;
    const res = https.post({
      url,
      headers: {
        Authorization: `Bearer ${getAccessToken(config)}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });
    if (res.code >= 300) {
      throw new Error(`RESTlet call failed: ${res.code} ${res.body}`);
    }
    return JSON.parse(res.body);
  };

  const execute = () => {
    const config = getConfig();
    const result = postToRestlet(config, { externalId: "ORD-1001", lines: [{ sku: "ABC-1", qty: 2 }] });
    log.audit("RESTlet response", JSON.stringify(result));
  };

  return { execute };
});
```

### cURL (raw HTTP)

For debugging. cURL can't sign the JWT; build `$ASSERTION` with another example.

`Terminal`:

```bash
# 1. Exchange a signed JWT for an access token.
#    $ASSERTION is a PS256 JWT signed with private.pem (build it with any of the other examples).
curl -s -X POST "https://1234567-sb1.suitetalk.api.netsuite.com/services/rest/auth/oauth2/v1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer" \
  --data-urlencode "client_assertion=$ASSERTION"

# Response: {"access_token":"eyJ...","expires_in":"3600","token_type":"Bearer"}

# 2. Call the RESTlet with the token. Reuse $ACCESS_TOKEN for every call
#    until shortly before expires_in runs out, then repeat step 1.
curl -s -X POST "https://1234567-sb1.restlets.api.netsuite.com/app/site/hosting/restlet.nl?script=customscript_order_sync&deploy=customdeploy_order_sync" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"externalId":"ORD-1001","lines":[{"sku":"ABC-1","qty":2}]}'

# JWT header:  {"alg":"PS256","typ":"JWT","kid":"<certificate id>"}
# JWT payload: {"iss":"<client id>","scope":["restlets"],"aud":"<token url>","iat":<now>,"exp":<now+3600>}
```

## Troubleshooting

| Symptom | Cause |
|---|---|
| `invalid_scope` on the token call | Integration record lacks the *RESTlets* scope, or the JWT requests a scope the record doesn't have |
| `invalid_grant` or `invalid_client` | Client ID, certificate ID, or key doesn't match the mapping; certificate expired or revoked; server clock more than a few minutes off |
| `401 INVALID_LOGIN` on the RESTlet | Request went to the `suitetalk` host instead of `restlets`, or the token expired |
| `403` / `INSUFFICIENT_PERMISSION` | Deployment not Released, role not in Audience, or role lacks a permission the script uses |
| `SSS_MISSING_REQD_ARGUMENT` or "invalid script" | Wrong `script` or `deploy` query parameter |
| RESTlet receives an empty body | Missing `Content-Type: application/json` |
| DNS error, host not found | Account ID not lowercase with `_` changed to `-` |

## Security

- Store `private.pem` in a secret manager; never in Git or a container image.
- Use a separate key and certificate mapping per environment (sandbox, production).
- Rotate without downtime: map a new certificate alongside the old one, switch the app, then revoke the old mapping.
- If a key leaks, revoke its mapping immediately. Issued tokens stop working within about an hour.
- Never log the token request; it contains the signed assertion.
