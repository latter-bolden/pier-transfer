# Pier Transfer Protocol

## Overview
This protocol is designed to facilitate the transfer of an Urbit pier from one system to another over the network. It should support the following use cases:
- Transfering from one hosting provider to another
- Transfering from Port to a hosting provider
- Transfering from a hosting provider to a local Urbit box (e.g. Native Planet)

## Terminology
- **Origin**: the system where the pier is currently running
- **Target**: the system where the pier is being transfered to
- **Base endpoint**: Target URL at which transfers can be initiated
- **Body**: the body of the request, set to MIME type `application/json`
- **Error message**: In requests where the status is in the 400's, an error message can be set using the response body set to `application/json` in the field `errorMessage`

## General Requirements
- All requests must happen over `HTTPS`
- Pier archives must be tarred and gzipped in format (`tar.gz`)

## Protocol
The Origin system will always initiate the transfer and the proccess is as follows:

### Step 1: Origin initializes the session
Origin must issue a `POST` request to `https://{Base Endpoint}` with the following parameters set in Body:

- `patp`: the patp of the ship being transfered
- `pierSize`: the size of the archived pier in megabytes
- `sessionId`: a [UUID v4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) that can be used as an identifier for the session
- `checksum`: a [MD5](https://en.wikipedia.org/wiki/MD5) hash of the archived pier that will be uploaded

Optionally, the following field can be set:
- `webhookEndpoint`: an endpoint where Target can notify Origin of successful authorization if required

If `pierSize` is too large for Target, it should reply with status `422` and the Error Message set to "Pier size too large". Optionally, it can return
the max size accepted in megabytes using the field `maxPierSize`.

If Target wishes to accept the request, its response must include a `state` field which will be set to either `requires-auth` or `ready`. Depending on
how `state` is set, proceed to [Step 2a](#step-2a-target-requires-authorization) or [Step 2](#step-2-target-ready) respectively.

<br/>

### Step 2a: Target requires authorization
If `state` is set to `requires-auth`, the following field must also be set in Body:
- `authEndpoint`: the session specific URL where the user can go to authenticate.

After receiving this response, the Origin application should direct the user to the specified endpoint for authorization. In the case of a new
customer to a hosting provider, this is where account creation and payment should occur. 

After authorization is complete, the user should be redirected
to `https://{Base Endpoint}/transfer/{sessionId}/auth-complete`. If `webhookEndpoint` was supplied in the initial request, it must be `POST`'ed to with the query parameter `message` set to `auth-complete`. 

Origin should either detect the redirect or webhook message and issue a `GET` request to `https://{Base Endpoint}/transfer/{sessionId}`. Target should now respond with `state: "ready"` and Origin can proceed to [Step 2](#step-2-target-ready).

<br/>

### Step 2: Target ready
If Target responds with `state: "ready"`, the following fields must also be set in Body:
- `sessionId`: the session UUID that was originally passed by Origin
- `uploadEndpoint`: the URL where the pier should be uploaded to
- `supportContact`: an email or phone number where Target support can be reached if there's an issue with the upload
- `expiresAt`: the expiration time after which uploads will not be accepted, encoded in ISO format. It should be at least four hours and at most 24 hours after the initial request was received.

Optionally, the following fields may also be set in Body:
- `requestProxy`: Specifies that Target wishes for Origin to set it as a management proxy on the Urbit ID of the pier being transfered (boolean).
- `proxyAddress`: The address which target wants the management proxy to be set to.

Subsequent `GET` requests to `https://{Base Endpoint}/transfer/{sessionId}` should return this same body. After receiving these details, Origin can proceed to [Step 3](#step-3-origin-uploads-pier).

<br/>

### Step 3: Origin uploads pier
Origin must issue a `POST` request to the specified `uploadEndpoint` with the MIME type set to `multipart/form-data`. The following field must
be included in the form:
- `sessionId`: the session UUID that was originally passed by Origin
- `pier`: the file of the archived pier to be uploaded

Optionally, the following field can be included:
- `proxySet`: whether or not the managment proxy was set if requested.

If the request fails, Origin should retry two more times before giving up. If subsequent attempts fail, the `supportContact` can be provided to handle
remediation out-of-band.

<br/>

### Step 4: Target confirms upload
If the upload was processed correctly, Target must confirm the checksum provided in the initial request matches the uploaded pier archive. If it doesn't
match, Target should respond `400` with the Error Message set to `Checksum mismatch`.

If the checksums match, Target must respond with the following fields set in Body:
- `sessionId`: the session UUID that was originally passed by Origin
- `state`: set to `completed`

Subsequent `GET` requests to `https://{Base Endpoint}/transfer/{sessionId}` before the stated `expiresAt` time should return these fields as well. **Only after Origin receives a response with `state: "completed"` should responsibility for the pier assumed to be transfered.**

After receiving such a response, the Origin application should notify the user that the request succeeded and prevent any subsequent boots of that pier. If the pier archive is not removed from disk, it should be placed somwhere that makes clear it should not be run unless the user knows what they're doing.

<br/>

## Example Flow
Below is an example flow using this protocol to upload a pier from Port to Example Hosting (EH), a hosting provider. Port is aware that EH's Base Endpoint is `https://example-hosting.com/pier-transfer`.

## Step 0:
The user indicates that they'd like to initiate a pier transfer and selects Example Hosting as the provider they want to use. Port then archives the
pier and calculates its size and MD5 hash.

## Step 1:
Port issues a `POST` request to `https://example-hosting.com/pier-transfer` with the following request body:

```json
{
  "sessionId": "30b33030-e4ba-465c-9dde-7b75a34426b1",
  "patp": "~latter-bolden",
  "pierSize": 7500,
  "checksum": "670751727d10d3571169e99d7131da56"
}
```

EH responds:
```json
{
  "sessionId": "30b33030-e4ba-465c-9dde-7b75a34426b1",
  "state": "requires-auth",
  "authEndpoint": "https://example-hosting.com/pier-transfer/30b33030-e4ba-465c-9dde-7b75a34426b1/sign-in"
}
```

## Step 2:
Port opens a new window set to the provided endpoint to complete authorization. Port detects a redirect to `https://example-hosting.com/transfer/30b33030-e4ba-465c-9dde-7b75a34426b1/auth-complete`. 

Port then issues a `GET` request to `https://example-hosting.com/pier-transfer/transfer/30b33030-e4ba-465c-9dde-7b75a34426b1` and EH responds with:
```json
{
  "sessionId": "30b33030-e4ba-465c-9dde-7b75a34426b1",
  "state": "ready",
  
  "uploadEndpoint": "https://example-hosting.com/upload/AJD-39CJ",
  "supportContact": "support@example-hosting.com"
  "expiresAt": "2022-12-22T05:00:00.000Z"
}
```

## Steps 3 and 4
Port uploads the pier to `https://example-hosting.com/upload/AJD-39CJ` and receives a response of:
```json
{
  "sessionId": "30b33030-e4ba-465c-9dde-7b75a34426b1",
  "state": "completed",
}
```
Port then notifies the user that the transfer succeeded and removes the pier from it's list of bootable ships.
