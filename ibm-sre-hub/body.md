{{ "1" | chapter("Overview") }}

During one of our weekly standups, one of the senior devs mentioned how it would be cool to have some type of way to display a lot of our product data using a spreadsheet-like format. As a SRE intern, he wasn't aware of my previous experience creating websites for fun just like this one, so I decided it would be a fun exercise and a great contribution to the team if I initialized the creation of such a tool to be taken over and developed. 

{{ "1.1" | subchapter("The Basics") }}

Since I was the only one on this team with recent webdev experience, I was able to choose my framework. I chose astro due to my familiarity with it, its ease of use, and the fact that react has a ton of open source templates so I don't need to spend my time designing components. 

Architecturally, I wanted to make it as **coupled** with the rest of our services as possible. This would allow the team to manage the website as minimally as possible, ensuring minimum points of failure. For authentication, it uses our ISV (IBM Security Oauth 2.0) groups, ensuring permissions with accessing the website would be controlled at the same time as permissions for any of our other services.

Furthermore, the data retrieved through many different microservices consists of a lot of info, but I decided a no-database version of the application would be better again to allow permission control to be coupled with our ISV. We have many environments, some of which consist of only test customers, but we also have environments with real production customers. Since the product (Defender Storage) is a security application, ensuring only certain people can access certain data from these tenants is enforced through our ISV groups again. This introduced some problems, which I will mention as well as the way that I decided to solve it.

{{ "1.2" | subchapter("Designing the Website") }}

It took many iterations of design, usually dealt with after a functional component was already made, but I think at the end it looked really good. 

<insert images>

{{ "2" | chapter("Architecting the API Request Logic") }}

{{ "2.1" | subchapter("Handling the basic data fetching") }}

The main customer (will refer to as tenant from now on) info is served as a list from the /tenants endpoint of one of our microservices. The old tool that this one was trying to replace had implemented each endpoint as a seperate cli tab, so the main architectural necessity was to have a way to present the useful information all at once. However, all of the data that was useful came in many many different formats; some were just a dictionary of tenants as the key, some had the tenant specific info nested within them, and others only returned the info of one tenant at a time (this caused a lot of issues, see 2.2). Also there wasn't much documentation as the last person to work on the tool had switched teams, but luckily there was someone on the team was maintaining it and gave me a big start on where to find things. From there, it was a lot of trial and error, manually `curl`-ing endpoints with generated authorization keys, looking and asking my team what data is useful, and then finding efficient methods of extracting the data into a proper list of tenants with all the necessary info attached.


Once I had that figured out, I then had a problem of dealing with the authentication methods. For each environment, there is a seperate Oauth server that is used, so I created a static yaml file that stored the data per-env as a dictionary of dict\[env\]. Then, since the logic for combining the data was run in the astro backend node server (not client side), I could just process that yaml directly into an object and access it there. Now was the part of dealing with the authorization. It used a device-code Oauth 2.0 method, but I found the IETF standard documentation to be extremely easy to read and understand ((here)[https://datatracker.ietf.org/doc/html/rfc8628]). 

{{ "2.2" | subchapter("OAuth 2.0 Device Authorization Grant Implementation") }}

OAuth 2.0 Device Authorization Grant (RFC 8628) was the backbone of authentication for this tool. It was selected due to ISV's support for this protocol and its suitability for headless devices or browser-based flows without direct access to user credentials.

The flow consists of several steps and maps directly to the code implementation:

**Step A & B – Requesting device and user code**

The frontend sends a request to the `/api/tenant_authLink.ts` endpoint. This hits ISV with the client ID (per environment), returning a `device_code`, `user_code`, and a `verification_uri_complete`. This is rendered as a toast with a link for the user to visit and complete the authentication.

**Step C & D – User completes authorization**

The user opens the provided link, logs in with their IBM ID, and is added to any ISV groups as per their permissions. This happens outside the site, and the frontend does not poll or track this state.

**Step E – Polling for token**

The backend starts polling ISV with the device code using `/api/tenant_authCookie.ts`. This polling lasts 60 iterations, with a short delay between each. Once the user completes the login, ISV returns an access token (`defenderToken`), which is stored in an `HttpOnly` cookie scoped to the environment.

**Step F – Token usage**

After token acquisition, the backend fetches protected endpoints using the bearer token. If the request succeeds, the session is stored. If the token is expired or invalid (e.g., 401), the backend deletes the cookie and restarts the flow.

**Security consideration**

Since the polling and token exchange are done entirely on the backend, there's no risk of exposing tokens client-side. The `defenderToken` is never visible in JS and is only attached to internal fetches from backend routes.

**Edge cases handled**

- Users not added to ISV groups return 500s — retried with optional tenant override.
- Tokens expire — backend clears cookie and retries.
- Multiple environments — each has its own isolated token/cookie based on `envKey`.
- Missing or malformed `device_code` — handled with schema validation and hard failure.

<insert image>

{{ "2.3" | subchapter("Session Handling and Caching") }}

Since the application never uses a database, all session information is ephemeral and coupled to the authentication cookie (`defenderCookie_<env>`). This cookie is set only on successful token retrieval and cleared aggressively when stale. Every fetch request either continues using the session or re-triggers a new device flow.

To avoid hammering endpoints and reauthenticating unnecessarily, tenant data is encrypted and cached in local storage using a per-environment key. This allows fast refresh and reduces load on microservices while maintaining security boundaries.

{{ "2.4" | subchapter("Data Normalization and Display") }}

Displaying all tenant data at once meant unifying formats across disparate APIs. Each endpoint had to be massaged into a standard `Tenant` shape before display. This included:

- Normalizing field names
- Converting nested and per-tenant responses into flat records
- Appending missing metadata from fallback endpoints
- Deduplicating or merging partial records

The result was a dense, spreadsheet-like table that unified operational, metadata, and cluster information into one screen with filters, sorting, and search. Most importantly, it was reactive to environment and tenant scope. However...

{{ "2.4" | subchapter("Issue with 1 endpoint: License Information") }}

One big issue was one of our endpoints was queried with the request header ?=tenantId, and each request required both switching the user group to a different group (the one that had access to the tenant) and then querying the endpoint. Because of this, it took anywhere from 3-30 seconds per request, with no way of parellizing the queries. My solution was to decouple this datapoint from the rest of them, loading it on request rather than during the main reload request. It also encrypts the data at rest with sha256 before putting it in the localstorage, but when loading it it grabs these datapoints seperately. I also implemented a progress bar to go along with this, giving the option to load all the data once as this data was something that doesn't change frequently. 

{{ metadata.images.image1 | image_process("alt text", "med") }}

{{ "1.3" | subchapter("subtitle") }}

{% set html_info %} 
t 
h 
i 
s
{% endset %}

{{ "title" | rawhtml(html_info) }}