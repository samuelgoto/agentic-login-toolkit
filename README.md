Author: @samuelgoto
Date: Jan 12, 2026
Status: early draft

# The `<federation>` element

TL;DR; This is a proposal to allows website authors to declare to agentic browsers (e.g. LLM-powered browser actuation) that a federated login option (e.g. social login) is available so that it can be used as a tool (rather than UI actuation). 

## Problem Statement

In LLM-powered Agentic browsers, many user journeys involve logging in to websites.  As with most of the LLM-powered actuation, the LLM has a baseline understanding using statistical models that allows it to click on links and fill forms to assist the user through the process. As much as user agents are and should develop as many heuristics as possible to retrofit into the existing content on the Web, heuristics are, by design, unreliable and are expected to have lower precision and quality than structured content that is opted-into by website owners.

Currently, login forms are marked up with a combination of low-level opaque primitives in browsers, which prevents browsers (specially agentic ones) to offer high-level / structured ways to log users in (e.g. an account chooser). Federated login, specifically, is generally presented as a series of "Sign-in with IdP" buttons that are typically marked up as a `<a>`, a `<form>` or javascript, such as `window.open()` or a `window.location.href`. For example:

```html
<a href="https://idp.example/oauth?...">
  Sign-in with IdP
</a>
```

The problem here is that, because this is just any other combination of low level HTML tags, the agentic browser can't infer federated login is offered as an option.

What’s the best way that website authors can annotate their page to make agentic browsersbetter aware of their federated login flows?

## Goals

There are many conflicting goals that we are navigating, but here are a few that we have found useful to constrain the solution space:

- Must allow authors to provide federated login semantics to agentic browsers
- Must degrade gracefully when browsers don’t support it
- Must support wrapping existing deployed patterns of federated login on the Web
- Must have a well defined accessibility contribution
- Must work outside of agentic browsers (makes testing/developing/maintaining much easier)
- Must allow authors to control the semantics in non-agentic browsers (allows authors to opt-out of deployment in non-agentic browser)

## Proposal

The proposal is to create an element that can wrap existing markup that allows the website author to declare what is the equivalent Federated Credential Management API request.

The `<federation>` element’s semantics are that, when clicked, it executes a credential management API call to FedCM’s active mode. The element’s display (e.g. CSS) is controlled by its inner children.

Before:

```html
Welcome to my website!

<a href="https://idp1.example/oauth?">
  Sign-in with IdP 1
</a>

<a href="https://idp2.example/oauth?">
  Sign-in with IdP 2
</a>
```

After:

```html
<federation onlogin="callback"
  clientId="1234"
  configURL="https://idp.example/config">
  <a href="https://idp1.example/oauth?...">
    Sign-in with IdP 1
  </a>
</federation>

<federation onlogin="callback"
  clientId="456"
  configURL="https://idp.example/config">
  <a href="https://idp2.example/oauth?...">
    Sign-in with IdP 2
  </a>
</federation>
```

Having a `<federation>` element that has real semantics makes it much easier for developers to implement it, because they can test and prototype its usage locally in a traditional browser window (as opposed to ARIA and microdata, which requires testing with a screen reader and a search engine respectively).

However, it should also be possible for the author to choose to only make the markup available for agentic browsers, so a parameter called `allow="agentic"` is introduced to allow-list only agentic use.

`<federation>` has a `generic` ARIA role (similar to `<span>`’s contribution to ARIA) and relies on its inner children to set up the right ARIA roles.

Here are the attributes of the <federation> element:

- clientId
- configURL
- fields
- params
- loginhint
- domainhint
- onlogin (callback)
- allow (deployment control)

# Future Work and Forwards Compatibility

We’d expect agentic login to incrementally need more and more declarative metadata about the authentication mechanisms that are available on the page.

Specifically, we’d expect that new elements would be introduced to support passkeys, usernames/passwords, SMS and email OTPs and other emerging forms of digital identities, such as verified email addresses and government issued digital identities.

There are some passkeys flows, specifically, that look like buttons, and could be easily translated to an equivalent <passkey> element:

Before:

```html
<span onclick="navigator.credentials.get({
  publicKey: ...})">
  Sign-in with Passkeys
</span>
```

After:

```html
<passkey
   onselection="callback"
   challenge="1234"
   rpId="example.com"
   userVerification="preferred"
   timeout="60000">
  <span onclick="navigator.credentials.get({
    publicKey: ...})">
    Sign-in with Passkeys
  </span>
</passkey>
```

Once `<federation>` and `<passkey>` are introduced, we’d be able to introduce a `<login>` element that works like a `<select>` and `<option>` elements and displays an account chooser as an inline element.

Something along the lines of:

Before:

```html
Login to my website!

<span onclick="navigator.credentials.get({
  publicKey: ...})">
  Sign-in with Passkeys
</span>

<a href="https://idp.example/oauth?">
  Sign-in with IdP
</a>

<form>
  Or enter your username/passwords:
  <input autocomplete=”username”>
  <input type=”password”>
</form>
```

After:

```html
<login onselection=”callback”>

  <passkey
     challenge="1234"
     rpId="example.com"
     userVerification="preferred"
     timeout="60000">
    <span  onclick="
       navigator.credentials.get({
          publicKey: ...})">
      Sign-in with Passkeys
    </span>
  </passkey>

  <federation
    clientId="1234"
    configURL="https://idp.example/config">
    <a href="https://idp.example/oauth?...">
      Sign-in with IdP
    </a>
  </federation>

  <form>
    Or enter your username/passwords:
    <input autocomplete=”username”>
    <input type=”password”>
  </form>

</login>
```

Conditional mediation requests for passkeys should also work, because there is sufficient annotation in the form elements in the form of an autocomplete=”webauthn” request.

Before:

```html
Login to my website!

<script type=”module”>
const passkey = 
  await navigator.credentials.get({
    Mediation: “conditional”,
    publicKey: ...
  });
</script>

<form>
  Or enter your username/passwords:
  <input autocomplete=”username”>
  <input type=”password” 
    autocomplete=”webauthn”>
</form>
```

After:

```html
<login onselection=”callback”>

  <script type=”module”>
  const passkey = 
    await navigator.credentials.get({
      Mediation: “conditional”,
      publicKey: ...
    });
  </script>

  <form>
    Or enter your username/passwords:
    <input autocomplete=”username”>
    <input type=”password” 
      autocomplete=”webauthn”>
  </form>

</login>
```

The introduction of a new `<login>` element that has UI semantics requires both `<federation>` and `<passkeys>` to be introduced, as well as developer activation, so it is not an immediate goal of this proposal, and is left as a future exercise that we have intentionally tried to design `<federation>` in a forwards compatible way.

## Alternatives Under Consideration

There are two dimensions to be considered here with various options in each one: (a) how to encode semantics and (b) which ontology to use.

Here are a few ones that I’m aware of:

- Serialization
  - microformats
  - RDFa
  - JSON-LD
  - Use the `<data>` element
  - Use the `data-*` attribute
  - The `autocomplete` attribute
  -- Something entirely new?
- Semantics
  - Activity Streams
  - Should we use https://schema.org/RegisterAction instead?
  - The `autocomplete` taxonomy
  - Something entirely new?
  - Something new?
  - Invent a new attribute
  - WebMCP: https://github.com/webmachinelearning/webmcp/issues/22
 
Here are a few compelling variations that we are actively exploring:

### ARIA `role="login"`

This is a variation to augument `role` with an additional landmak, `login`, akin to [`search`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Roles/search_role):

```html
<span role="login">
  <!-- how would we encode the rest of the FedCM parameters in the ARIA parameters? maybe that's not right? -->
  Sign-in with X
</span>
```
### Microdata

This proposal is to introduce to LoginAction a property called `federation` which describes what the FedCM request would be.

For example:

```html
<div itemscope itemtype="https://schema.org/LoginAction">
  <data itemprop="federation" 
    value="client_id=\"1234\", config_url=\"https://idp.example/fedcm.json\"" />
  <button>Sign-in with X</button>
</div>
```

### JSON-LD

This proposal is to introduce to LoginAction a property called `federation` which describes what the FedCM request would be.

For example:

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "LoginAction",
  "federation": {
    "providers": [{
      "configURL": "https://idp.example/config.json",
      "clientId": "1234",
      "nonce": "4567",
      "fields": ["email", "name", "picture"],
     }]
  },
}
</script>
```

The invocation of the action occurs via a DOM event:

```javascript
document.addEventListener("action", ({type, federation: {token}}) => login(token));
```

### Mediation: `conditional`

In this variation, we use the `mediation="conditional"` parameter to let the agent operate in the unresolved promise.

```javascript
const {token} = await navigator.credentials.get({
  mediation: "conditional",
  identity: { /** ... params ... */ }
});
```

### `<permission type="login">`

We could extend the [PEPC element](https://github.com/WICG/PEPC/blob/main/explainer.md) to introduce a `type="login"` parameter.

```html
<permission type="login" federation="clientId='1234', configURL='https://idp.example/config.json'">
   <a href="https://idp.example/oauth?...">Sign-in with IdP</a>  
</permission>
```

### `meta` tags

In this variation we’d use the <meta> tag disassociated with the element to be clicked.

```html
<meta http-equiv="Federated-Authentication" 
  content="client_id=\"1234\", config_url=\"https://idp.example/fedcm.json\""
>
<script>
document.addEventListener("login", ({token}) => login(token));
</script>
```

## Alternatives Considered

### Overload WWW-Authenticate

In this variation we’d support a declarative request made via HTTP headers, like WWW-Authenticate or introduce a few one:

```
WWW-Authenticate: Federated; client_id="1234", config_url="https://idp.example/fedcm.json"
```

Cons:

- Requires RPs to redeploy their servers
- WWW-Authenticate is blocking (and because of that, we think, poorly adopted)

### Login page discovery

When agents get to a website and want to login the user to it, they need to first find the login page in the first place. This often involves a series of heuristics and computer vision, but the developer can help the agent find it with the following convention:

```html
<link rel="login" href="login.html">
```
