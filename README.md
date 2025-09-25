# SiteSage AI Widget

SiteSage is an application that enhances ticket systems with certain AI features. There is a backend and front-end / app component. SiteSage focuses 100% on the AI part, and works in close contact with our customers, to help them create a customized AI setup that works for them.

Enhancing is done via webhook that listens in on the incoming requests, and performs actions via the ticket system API. As part of this, a reply suggestion is also generated, which is later available through the app.

The app is injected into the ticket system, this is usually the hard part.

Ideally the app should be available as a widget within the ticket system. It needs to be injected as an iframe and communication between the ticket system platform and SiteSage app usually happen via `postMessages` .

## Widget Installation Parameters

- api_token: string
- app_url: string
- backend_url: string

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

## Getting started with SiteSage

If you're looking for an easy way to integrate SiteSage into your ticket system the `sitesage ticket script` is a light weight script that exposes a single helper class to handle the messaging to and from the SiteSage iframe.

```html
// coming soon
<script src="..."></script> 
<script>
	const iframe = document.getElementById("{the-sitesage-iframe}")
	SiteSage.init(iframe);
	SiteSage.load(data);
</script>
```
