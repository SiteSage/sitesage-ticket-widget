# SiteSage AI Widget

SiteSage is an application that enhances ticket systems with certain AI features. There is a backend and front-end / app component. SiteSage focuses 100% on the AI part, and works in close contact with our customers, to help them create a customized AI setup that works for them.

Enhancing is done via webhook that listens in on the incoming requests, and performs actions via the ticket system API. As part of this, a reply suggestion is also generated, which is later available through the app.

Ideally the app should be available as a widget within the ticket system. It needs to be injected as an iframe and communication between the ticket system platform and SiteSage app usually happen via `postMessages`.

See the [Getting started](#getting-started-with-sitesage) section for easily managing this with the `SiteSage Web Component`.

## Widget Installation Parameters

- api_token: string
- app_url: string
- backend_url: string

## Getting started with SiteSage

If you're looking for an easy way to integrate SiteSage into your ticket system the `sitesage ticket script` is a light weight script that exposes a `sitesage-widget` Web Component to handle the messaging to and from the SiteSage iframe.

Below is a minimal example.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <script type="module" src="./dist/sitesagejs.iife.js"></script>
</head>
<body>
    <div class="sitesage-container">
        <sitesage-widget
            app_url="install-param-app_url"
            backend_url="install-param_backend_url"
        />
    </div>
</body>
</html>


<script>

    window.addEventListener("DOMContentLoaded", () => {

        const sitesage = document.querySelector("sitesage-widget");

		// Update the app language and agent name. Role *must* be 'agent'.
        sitesage.attributes.user.value = JSON.stringify({role: "agent", name: "John Doe", language: "da-DK"});
		// Let the app know about the current ticket. It is not required to reload the app when navigating to a new ticket. Simply update this value.
        sitesage.attributes.ticket.value = JSON.stringify({message_id: 123, conversation_id: 456});

		// Insert text into the ticket area. Specific to your ticket system.
        sitesage.addEventListener("texteditor:insert", (event) => {
            console.log("texteditor insert", event.detail.text);
        });

		// To avoid exposing the customer API key, please wrap network requests in a proxy on the server.
        sitesage.addEventListener("proxy:request", async (event) => {
            const {request, respondWith, error} = event.detail;
            try {
                const data = await server_side_proxy(request);
                const response = new Response(JSON.stringify(await data.json()), {
                    status: data.status,
                    headers: new Headers(data.headers),
                });
                respondWith(response); // expexts a regular response object.
            }catch(e) {
                console.warn(e);
                error("Proxy Failure");
            }
        })
    });

    /**
     * This code should not run on the client side as it contains secrets.
     * It is only here to show how to handle the proxy request and serves as an example.
     */
    function server_side_proxy(request) {
        const SECRET_KEY_FROM_SERVER = "customer-sitesage-api-key";

		// SiteSage sends the Authorization Header in mustache template format: `Authorization Bearer {{sitesage_api_key}}`
		// replace the value of {{sitesage_api_key}} with the customer secret api key from installation parameter.
        const secureHeaders = request.headers.map(([key, value]) => {
            return [key, value.replace('{{sitesage-api-key}}', SECRET_KEY_FROM_SERVER)];
        });
        
        return fetch(request.url, {
            method: request.method,
            headers: secureHeaders,
            body: request.body,
        });
    }



</script>


<style>
    .sitesage-container {
        width: 600px;
        height: 320px;
    }
</style>
```

# Low-Level Message Types

If you are using the SiteSage Web Component there is no need to implement the following directly.

## Message Events

Communication between the ticket system platform and SiteSage app usually happen via `postMessages`  with these events

### Messages from SiteSage Iframe

**APP_READY**
The `APP_READY` is sent when the iframe is loading and are ready to receive the `APP_OPEN` event.
```ts
type: "APP_READY",
is_ready: true
```
**FETCH_REQUEST**
`FETCH_REQUEST` is a request for fetching a resource on the backend via proxy. Requests to the backend needs the `api_token` in the `Authorization` header.

The `id` is used to identify the response in `FETCH_RESPONSE` which should return a matching id.
```ts
type: "FETCH_REQUEST",
id: string,
url: string,
method: "POST" | "GET",
body: string,
headers: {
	["Content-Type"]:  "application/json",
}
```
**TICKET_AREA_INSERT**
`TEXT_EDITOR_INSERT` is a request for inserting data into the ticket editor area in the ticket system.
```ts
type:  "TEXT_EDITOR_INSERT",
content: string
```

### Messages to SiteSage Iframe

**APP_OPEN**
`APP_OPEN` will open the app with certain information about the specific ticket and agent using it. It is possible to send multiple `APP_OPEN` events and have the app update.

```ts
type: "APP_OPEN"
ticket_id: string | number,
agent_language?: string,
agent_name?: string,
customer_language?: string,
customer_email?: string,
```
**FETCH_RESPONSE**
`FETCH_RESPONSE` should be returned as a response to a `FETCH_REQUEST` event. (see proxy fetching).
```ts
type: "FETCH_RESPONSE",
id: string,
data: any,
headers: any
```

## Proxy Fetching

Proxy fetching allow the iframe to authenticate securely with the SiteSage backend, without needing to know about the `api_token` installation parameter that is injected into the `Authorization` header.

**Example:**
```ts
async function proxy_fetch(data) {
    // OBS: only allow proxying to known whitelisted endpoint(s).
    if( !data.url?.includes(backend_url) ) return; 
    
    const response = await fetch(data.url, {
        ...data.options,
        headers: {
            ...data.options.headers,
            'Authorization': `Bearer ${api_token}`,
        },
    });

    const contentType = response.headers.get('content-type') || '';
    const body = contentType.toLowerCase().includes('application/json')
        ? await response.json()
        : await response.text();

    return {id: data.id, ok: response.ok, body}
}
```

Then listen to incoming messages and call the functions like so:
**Example:**
```ts
window.addEventListener("message", async (event) => {
	// OBS: only allow traffic from known origin.
	if(event.origin !== app_url.origin) return;

	switch (event.data.type) {
		case "APP_READY":
			event.source.postMessage({
                type: 'APP_OPEN',
                ticket_id: ticketId,
                ... // other info
            }, app_url.origin);
			break;
	    case "TEXT_EDITOR_INSERT":
	        insert_text(event.data?.content) // handle insert
	        break;
	    case "FETCH_REQUEST":
	        const res = await proxy_fetch(event.data).catch(() => ({
	            id: event.data.id, type: "FETCH_RESPONSE", ok: false
	        }));
	        event.source.postMessage({...res, type: "FETCH_RESPONSE"}, app_url.origin); 
	    default:
	        break;
	}
```

