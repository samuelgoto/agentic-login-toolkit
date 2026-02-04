Author: @samuelgoto
Date: Jan 12, 2026
Status: early draft

> With a massive amount of guidance from Philip Jägenstedt (on relationship to `<search>`, `<main>`, `ARIA` and `microdata`), Jeffrey Yasskin (on relationship to `<geolocation>` and `<permission>`, as well as `JSON-LD`)  Khushal Sagar and Dominic Farolino (on relationship to `WebMCP`), Ryan Levering (on relationship to `schema.org`) and Christian Biesinger (on a variety of `HTML` design choices).

# Declarative FedCM: The `<federation>` element

TL;DR; This is a proposal to allows website authors to declaratively markup federated login options with a new element, `<federation>`, with a participating FedCM identity provider, as an alternative to the already existing imperative Javascript option. The `<federation>` element plays three roles: (a) it allows website authors to declaratively and ergonomically use federation, (b) it allows browsers to discover federated login option uniformally and finally (c) it serves as a necessary stepping stone towards a unified account chooser `<login>` element (see forwards compatibility section below).

## Problem Statement

Users of the web are increasingly using agentic browsers to complete their journeys. Many of these journeys involve logging in to websites with the user's passwords and federated accounts, the two most common login mechanisms on the web today.

To retrofit and log users in the existing content, agentic browsers have developed a statistical LLM model that gives them a broad (but non-deterministic) understanding of web pages and login forms, enough to allow them to click on links and fill forms to assist the user through the process. 

Unfortunately, the same statistical design choice that brings the broad generalization and coverage also brings lower precision and recall compared to structured / deterministic APIs that are opted-into by website owners.

Currently, login forms are marked up with a combination of low-level opaque primitives in browsers. Federated login, specifically, is generally presented as a series of "Sign-in with IdP" buttons that are typically marked up as a `<a>`, a `<form>` or javascript, such as `window.open()` or a `window.location.href`. For example:

```html
<a href="https://idp.example/oauth?...">
  Sign-in with IdP
</a>
```

The problem here is that, because this is just any other combination of low level HTML tags, the agentic browser infers the options for federated login statistically rather than deterministically, in an otherwise well-defined standardized protocol (typically OIDC and SAML).

Because of that, many user journeys that involve users logging in to websites with their federated accounts end up failing more often than not (in an unpredictable way).

Fortunately, a meaningful percentage of the federated login traffic has already been migrated to FedCM, especially for consumers, and APIs such as [IdP-initiated Request](https://github.com/fedidcg/idp-initiated) can further cover the remaining of the deployment setups. 

When the website uses FedCM to log users in via their federated accounts, the agentic browser is able to handle it as a structured tool, rather than as an unstructured statistical task.

For example, because FedCM is mediated by the browser, the agentic browser can be sure that it is not accidentaly creating an account on the website (by comparing the list of [`approved_clients`](https://w3c-fedid.github.io/FedCM/#dom-identityprovideraccount-approved_clients)), substantially making this operatoin safer for users.

```html
<a href="https://idp.example/oauth?..." 
  onclick="if (window.IdentityCredential) {navigator.credentials.get({identity: ...})} ">
  Sign-in with IdP
</a>
```

However, because FedCM has been deployed primarily as an imperative JS call (see snippet above), the browser isn't able to see it before the call is made.

Is there anything that websites authors can do to make agentic browsers aware that a federated login option is available through a FedCM request?

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

The `<federation>` element’s semantics are that, when clicked, it executes a credential management API call to FedCM’s `mode="active"`. 

The element’s display (e.g. CSS) is controlled by its inner children.

Before:

```html
<a href="https://idp.example/oauth?">
  Sign-in with IdP
</a>
```

After:

```html
<federation clientId="1234" configURL="https://idp.example/config">
  <a href="https://idp.example/oauth?...">
    Sign-in with IdP
  </a>
</federation>
```

Having a `<federation>` element that has real semantics makes it much easier for developers to implement it, because they can test and prototype its usage locally in a traditional browser window (as opposed to ARIA and microdata, which requires testing with a screen reader and a search engine respectively).

`<federation>` has a `generic` ARIA role (similar to `<span>`’s contribution to ARIA) and relies on its inner children to set up the right ARIA roles.

Here is the definition of the <federation> element:

```typescript
[
    Exposed=Window,
] interface HTMLFederationElement : HTMLElement {
    // FedCM request parameters
    [CEReactions, Reflect] attribute DOMString clientid;
    [CEReactions, Reflect, URL] attribute USVString configurl;
    [CEReactions, Reflect] attribute DOMString loginhint;
    [CEReactions, Reflect] attribute DOMString domainhint;
    [CEReactions, Reflect] attribute DOMString fields;
    [CEReactions, Reflect] attribute DOMString params;
    // Used when the IdP returns a token, rather than a redirect_to
    attribute EventHandler ontoken;
    readonly attribute DOMString token;
};
```

# Open Questions

- Can/should developers be able to control whether the `<federation>` element is a "semantics only" element (such as `<search>`) so that it can be deployed exclusively in agentic browsers (but not affect regular users?)? If so, how?

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

On a related but orthogonal note, I think agents would probably also benefit from a declarative link to the "login" page. Unclear if that necessarily needs to be handled by `<login>` or a `rel="login"` `<link>` would do.

```html
<link rel="login" href="login.html">
```

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

### <script type="federation">

```html
<script type="federation">
{
  clientID: "1234",
  configURL: "https://idp.example",
}
</script>
<script>
document.addEventListener("login", () => ...)
</script>
```

### ARIA `role="login"`

This is a variation to augument `role` with an additional landmak, `login`, akin to [`search`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Roles/search_role):

```html
<span role="login">
  <!-- how would we encode the rest of the FedCM parameters in the ARIA parameters? maybe that's not right? -->
  Sign-in with X
</span>
```

Open questions:

- How would we encode the FedCM parameters in ARIA?
- How do we throw a javascript event to return the result?
- What should be the ARIA role that the "login" role should have? Should it be a landmark?
- How do we handle non-conforming screen readers? How do we make it backwards compatible to unchanged screen readers?

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

Open Questions:

- See open questions about ARIA above

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
<script>
document.addEventListener("login", () => ...)
</script>
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

