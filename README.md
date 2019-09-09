# Cloud OAuth

> Website, Cloud-Connector-Addon and external service authentication.

This is unrelated to OHX installation authentication.

This service implements the OAuth Code Grant and OAuth Device flow in combination
with the website, which renders the login and token generation endpoint in combination with
[Google Firestore Auth](https://firebase.google.com/docs/auth).

Firestore tokens are exchanged into OHX OAuth tokens and can be revoked by the user.

Find all allowed services in `oauth_clients.json`:
The addon registry CLI tool, Amazon Alexa and the Cloud Connector Addon are important clients.

Looking at Amazon Alexa as an example, the procedure works like this:

* The Amazon Alexa App opens https://openhabx.com/auth?client_id={}&client_secret={}&state={}&redirect_url={}
* The user logs in via his openhabx.com credentials and confirms access.
* The https://oauth.openhabx.com/generate_token endpoint of this service is called to
  create a real access token and refresh token, which is persisted in the firestore database.
  A `code` is returned.
* The webpage redirects to the given `redirect_url`, appending the `code`.
* Amazon servers will now swap the `code` for the generated access token / refresh token tuple
  via the https://oauth.openhabx.com/token endpoint of this service.
* The user can enumerate and revoke refresh tokens on https://openhabx.com/access_tokens.

For the CLI it works a bit different:

* The user starts the CLI to upload an Addon to the registry.
  He/She is asked to log in and https://openhabx.com/auth?client_id={}&client_secret={}&state={} is opened.
  A redirect url is not given, because we are following the indirect OAuth device flow now.
* The CLI will poll https://oauth.openhabx.com/check_for_auth?state={}.
* When the user successfully logged in, and the https://oauth.openhabx.com/generate_token endpoint has been called,
  the `check_for_auth` endpoint will return a real access token and refresh token.
  The tokens are persisted in the users addon directory and used next time.

OAuth alike endpoints:

* `/authorize?<response_type>&<client_id>&<redirect_uri>&<scope>&<state>`
  response_type can be "code" for the OAuth code grant flow or "device"
  Works in tandem with the websites /auth page.
  - For the device flow it will return a json (device_code,user_code,verification_uri,...) with a verification_uri like below.
  - The code grant flow will not show a page itself, but will redirect to URL.
  
  URL: `https://openhabx.com/auth?<response_type>&<client_id>&<redirect_uri>&<scope>&<state>&<unsigned>`.
* `/token`: OAuth Code to token endpoint. Used by the code grant and device flow.
  Expects POST form data with `grant_type`, `client_id`, `device_code` or `code`.
* `/grant_scopes?<unsigned>&<scopes>&<code>`: *². POST form request;
  Returns a 60 min valid "code" that can be used for the token endpoint to retrieve access tokens.
  Called by the websites `/auth` page that will soon after redirect to a given "redirect_uri" with that code.

"unsigned" is a service generated JWT, but not yet signed. So it cannot be used as an access token yet. 

Management endpoints:

* `/revoke`: *¹. POST; Expects a json {client_id,client_secret,token}
* `/check_users`: *¹. Check for users that are marked as to-be-removed and remove them. To be called periodically.
* `/userinfo?<user_id>`: *¹. Firestore user information
* `/list_intermediate_tokens`: *¹. Lists all generated codes that are not yet exchanged into oauth tokens

*¹: For Google Service Account Authenticated requests

*²: For Firestore-Authenticated requests.

## Implementation details

A client usually navigates to `/authorize` on any oauth implementation and the login / grant UI is presented.

Because this implementation strictly separates API from UI, which is statically hosted, a different approach has been chosen.
One goal is, to not persist anything on `/authorize`. Although the endpoint is rate limited, it could be abused if
a lot of authorisation requests are made with no intention to *tokenize* those intermediate codes.
 
This implementation needed to consider a few possible security threads.

##### The UI should not be able to tamper with the original request (ie the requested "scopes") made on the `/authorize` endpoint.

The UI and embedded javascript are considered not safe.
The UI should not be able to grant more access (add "scopes") than the original request asked for.

In this implementation a preliminary token is generated on `/authorize`, which already encodes the requested data like sanitised scopes.

* The jwt token is not signed (so not valid as it is)
* It is compressed and encrypted with a key only known to this service.
* A hash of this token is created and used as "code".
* `/authorize` will redirect the client to the UI page which gets the tuple `(unsigned_token, code)`

The UI will call `/grant_scopes?<unsigned>&<scopes>&<code>` and only if the given *code* matches
`hash(unsigned)` a real access token with the intersection of the granted scopes and originally requested scopes
is generated and persisted for 5 minutes. From this moment on a client can call `/token` with the `code`.
 
On success the UI will now redirect the client to the `redirect_uri` address that was originally given to `/authorize`
together with `code`.

##### An attacker should not be able to use the UI endpoint  `/grant_scopes` to generate tokens

* This endpoint can only be called with a valid firebase Auth access token.
* The service expects `unsigned` to be encrypted with the key only known to this service.

## Code structure

The `main.rs` contains only environment initializing boilerplate code, like preparing the logger.
The rocket web-framework instance with all runtime states (redis connector, firebase session, credentials) is
created in `lib.rs` within `create_rocket`. The actual http routes are stored in `routes.rs`.

Error handling within routes happens via a custom responder type `MyResponder`, defined in `responder_type.rs`.
All routes must return `Result<..., MyResponder>`.
`MyResponder` implements `From` traits (for example for `serde_json::Error`) to allow to use "?" within route methods. 

## Building from sources

Because a slim container is preferred, the binary is compiled with [musl](https://www.musl-libc.org/)
instead of libc to get a fully static, self-contained binary.
(Check with `ldd target/x86_64-unknown-linux-musl/debug/cloud-vault`.)
You need the "x86_64-unknown-linux-musl" rust target on rust nightly to be installed.
Use `rustup toolchain nightly` to install rust nightly and `rustup target add x86_64-unknown-linux-musl` to add the target. 

## Test run

Start with `RUST_LOG=info cargo run` to see the full logs.

## Deployment for the Cloud-Deployment team

Call `./build-and-deploy.sh` which builds in release mode and uploads a snapshot of the directory to Google Cloud Build.
The resulting container is deployed to G- Cloud Run.

* If the domain mapping got lost, restore it by calling `gcloud-domain-map.sh`.
* If the cron jobs are lost, restore those by calling `gcloud-cron-setup.sh`.