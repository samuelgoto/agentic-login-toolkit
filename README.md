Author: @samuelgoto
Date: Jan 12, 2026
Status: early draft

> With a massive amount of guidance from Philip Jägenstedt (on relationship to `<search>`, `<main>`, `ARIA` and `microdata`), Jeffrey Yasskin (on relationship to `<geolocation>` and `<permission>`, as well as `JSON-LD`)  Khushal Sagar and Dominic Farolino (on relationship to `WebMCP`), Ryan Levering (on relationship to `schema.org`) and Christian Biesinger (on a variety of `HTML` design choices).

# The `<login>` element

TL;DR; This is a proposal to allows website authors to use an inline `<login>` element to wrap their "login" links typically found on the top right corner. `<login>` renders like an `<a>` wrapping its inner content and opens a mediated **modal** dialog when clicked on with all login options available with a corresponding [Credential Management API](https://developer.mozilla.org/en-US/docs/Web/API/Credential_Management_API). The Credential Management call is constructed according to the options declared inline with the introduction of a declarative `<federation>` element, a `<passkey>` element and attributes to control passwords. The declarative `<login>` element allows browsers to pull login out of the content UI and into the common and interoperable browser UI assist users in ways that can't be done in userland (e.g. re-use preferences across sites, display in the URL bar and in agentic browser flows).

```html
<login onselect="login()">
  <passkey challenge="1234" rpId="example.com" userVerification="preferred" timeout="60000"></passkey>
  <federation clientId="1234" configURL="https://idp1.example/config"></federation>
  <federation clientId="5678" configURL="https://idp2.example/config"></federation>
  login
</login>
```

# Problem Statement

One of the most common patterns on the Web is to allow users to login to websites.

Unfortunately, lacking browser support, login has been constructed entirely on top of the browser using low level primitives, such as `<form>` for passwords/passkeys and `<a>` for social login, the two most common authentication mechahnisms, in what's commonly called the "NASCAR flag" UI (called so because it looks like an area filled with commercial brands).

When users interact with the NASCAR flag, they have to guess what to use between the various options: do I have a passkey? a password? or did I click in one of these social login options before?

Because the NASCAR flag is implemented in userland, it can't (by design) reconcile and unify across the various login methods, leading to confusion and friction at best and account duplication at worst.

Fortunately, browsers have been able to mediate more and more of the login flows, starting from some of the earliest attemps with [BasicAuth](https://en.wikipedia.org/wiki/Basic_access_authentication) but more recently with the advent of APIs such as [WebAuthn](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API) (a strong alternative to passwords), [WebOTP](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API) (for verifying phone numbers), [FedCM](https://developer.mozilla.org/en-US/docs/Web/API/FedCM_API) (for federation), [Digital Credentials](https://www.w3.org/TR/digital-credentials/) (for government-issued IDs) and [Email verification Protocol](https://github.com/WICG/email-verification-protocol) (for verifying email addresses), all exposed via the [Credential Management API](https://developer.mozilla.org/en-US/docs/Web/API/Credential_Management_API).

One recent step browsers are taking to unify these flows is via the [`immediate mediation`](https://github.com/w3ctag/design-reviews/issues/1092), which allows websites to get an account chooser that unifies across passwords/passkeys and social login.

However, because `immediate mediation` is an imperative API call, the browser can't use it outside of its content area, for example in a common browser UI area (e.g. the URL bar) or in agentic flows (e.g. when an LLM is helping the user login).

To further unify and reconcile across authentication mechanisms, would it be possible to create a declarative browser-mediated login flow that websites could use?

# Goals

There are many conflicting goals that we are navigating, but here are a few that we have found useful to constrain the solution space:

- Must cover the most common login mechanisms, specifically passwords/passkeys and federation
- Must allow website authors to provide declarative login semantics to browsers (to allow use outside of the content area, e.g. the url bar or agentic browsing)
- Must be able to be retrofited into existing websites (e.g. support feature detection)

# Proposal

The proposal is to introduce `<login>` element (along with a `<federation>` and `<passkey>` elements and more to follow) that can declaratively describe an imperative Credential Management API call.

The `<login>` element's semantics are simliar to `<a>`: it is an inline element that renders its inner contents and performs an action `onclick`.

The intention is to replace the typical "login" links that show up on the top right corner of websites with the following:

```html
<login onselect="login()">
  <passkey challenge="1234" rpId="example.com" userVerification="preferred" timeout="60000"></passkey>
  <federation clientId="1234" configURL="https://idp1.example/config"></federation>
  <federation clientId="5678" configURL="https://idp2.example/config"></federation>
  login
</login>
```

When clicked, a modal dialog is shown, with a browser mediated unified account chooser that contains all of the options specified by the developer.

# Sequencing

There is an overall belief that, while this seems like a plausible end state, it is likely that we can't quite reach it from where we are: [`immediate mediation`](https://github.com/w3ctag/design-reviews/issues/1092) is still unresolved and the developer demand for a unified account chooser is still in its infancy.

Agentic browsing, on the other hand, is testing the limits of the Web Platform and is currently strugging to log users into websites safely.

One way that occurred to us that we could kick start this process is to wrap the individual options in the NASCAR flag, rather than the NASCAR flag as a whole.

So, for example, for passkeys buttons:

```html
<login onselect="login()">
  <passkey challenge="1234" rpId="example.com" userVerification="preferred" timeout="60000"></passkey>
  <a onclick="navigator.credentials.get({publicKey: ...})">Sign-in with a Passkey</a>
</login>
```

And the following for social login buttons:

```html
<login onselect="login()">
  <federation clientId="1234" configURL="https://idp1.example/config"></federation>
  <a onclick="navigator.credentials.get({identity: ...})">Sign-in with IdP</a>
</login>
```

This would allow us to introduce `<login>` and `<federation>` for each of these individual mechanisms while working towards a unified UI for all of them.

# Open Questions

- Can/should developers be able to control whether the `<federation>` element is a "semantics only" element (such as `<search>`) so that it can be deployed exclusively in agentic browsers (but not affect regular users?)? If so, how?

# Alternatives Under Consideration

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

## <script type="federation">

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

## ARIA `role="login"`

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

## Microdata

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

## JSON-LD

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

## Mediation: `conditional`

In this variation, we use the `mediation="conditional"` parameter to let the agent operate in the unresolved promise.

```javascript
const {token} = await navigator.credentials.get({
  mediation: "conditional",
  identity: { /** ... params ... */ }
});
```

## `<permission type="login">`

We could extend the [PEPC element](https://github.com/WICG/PEPC/blob/main/explainer.md) to introduce a `type="login"` parameter.

```html
<permission type="login" federation="clientId='1234', configURL='https://idp.example/config.json'">
   <a href="https://idp.example/oauth?...">Sign-in with IdP</a>  
</permission>
```

## `meta` tags

In this variation we’d use the <meta> tag disassociated with the element to be clicked.

```html
<meta http-equiv="Federated-Authentication" 
  content="client_id=\"1234\", config_url=\"https://idp.example/fedcm.json\""
>
<script>
document.addEventListener("login", ({token}) => login(token));
</script>
```

# Alternatives Considered

## Overload WWW-Authenticate

In this variation we’d support a declarative request made via HTTP headers, like WWW-Authenticate or introduce a few one:

```
WWW-Authenticate: Federated; client_id="1234", config_url="https://idp.example/fedcm.json"
```

Cons:

- Requires RPs to redeploy their servers
- WWW-Authenticate is blocking (and because of that, we think, poorly adopted)

