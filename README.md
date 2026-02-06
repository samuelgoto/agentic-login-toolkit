Author: @samuelgoto
Date: Jan 12, 2026
Status: early draft

> With a massive amount of guidance from Philip Jägenstedt (on relationship to `<search>`, `<main>`, `ARIA` and `microdata`), Jeffrey Yasskin (on relationship to `<geolocation>` and `<permission>`, as well as `JSON-LD`)  Khushal Sagar and Dominic Farolino (on relationship to `WebMCP`), Ryan Levering (on relationship to `schema.org`) and Christian Biesinger (on a variety of `HTML` design choices).

# The `<login>` element

TL;DR; every website has to create their own login flow, leading to an inconsistent, fragmented, inneficient and cumbersome user experience. This is a proposal to allow website authors to introduce an inline `<login>` element to wrap their "login" links typically found on the top right corner on their pages and provide a browser mediated and unified login flow across all authentication mechanisms (notably passwords, passkeys and federation) across every website. `<login>` renders like an `<a>` wrapping its inner content and opens a mediated **modal** dialog when clicked on with all login options available with a corresponding [Credential Management API](https://developer.mozilla.org/en-US/docs/Web/API/Credential_Management_API). The Credential Management call is constructed according to the options declared inline declaratively with a new `<credential>` element, which can represent all of the supported credential types. In addition to the user experience benefits, the declarative `<login>` element allows browsers to pull login out of the content area into the browser area to, for example, re-use preferences across sites, display in browser UI (e.g. the URL bar) and discover login options in agentic browser flows.

```html
<login onselect="login()">
  <credential type="publickey" 
      challenge="1234"
      rpId="example.com"
      userVerification="preferred"
      timeout="60000">
  </credential>
  <credential type="federated"
    clientId="1234"
    configURL="https://idp1.example/config">
  </credential>
  <credential type="federated"
     clientId="5678"
     configURL="https://idp2.example/config">
  </credential>
  <a href="login.html">login</a>
</login>
```

This gets transpiled into the following in supporting browsers:

```html
<a onclick="navigator.credentials.get({publicKey: ..., identity: ...}).then(login)">
  <a href="login.html">login</a>
</a>
```

And degrades gracefully to the following in browsers without native support:

```html
<span>
  <a href="login.html">login</a>
</span>
```

# Problem Statement

One of the most common patterns on the Web is to allow users to login to websites.

Login typically starts with a "login" link, commonly available on the top-right corner of websites which usually leads to what's known as the NASCAR flag UI:
<img src="https://github.com/user-attachments/assets/e4f133c4-66c6-4f8f-9552-b7cc99d118ec" />
When users interact with the NASCAR flag UI, they have to guess what to use between the various options: do I have a passkey? a password? or did I click in one of these social login options before?

Because the NASCAR flag is implemented in userland, it can't (by design) reconcile and unify across the various login methods, leading to confusion and friction at best and account duplication at worst.

Fortunately, browsers have been able to mediate more and more of the login flows, starting from some of the earliest attemps with [BasicAuth](https://en.wikipedia.org/wiki/Basic_access_authentication) but more recently with the advent of modern APIs such as [WebAuthn](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API) (a strong alternative to passwords), [WebOTP](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API) (for verifying phone numbers), [FedCM](https://developer.mozilla.org/en-US/docs/Web/API/FedCM_API) (for federation), [Digital Credentials](https://www.w3.org/TR/digital-credentials/) (for government-issued IDs) and [Email verification Protocol](https://github.com/WICG/email-verification-protocol) (for verifying email addresses), all exposed via the [Credential Management API](https://developer.mozilla.org/en-US/docs/Web/API/Credential_Management_API).

While, for the most part, each of these credentials have been deployed independently, they have been deliberately designed as Credential Management credential types with the hope that we would be able to unify them at some point. One recent such step is via the [`immediate mediation`](https://github.com/w3ctag/design-reviews/issues/1092) proposal, which allows websites to get an account chooser that unifies across passwords/passkeys and social login.

However, because `immediate mediation` is an imperative API call, the browser can't use it outside of its content area, for example in a common browser UI area (e.g. the URL bar) or in agentic flows (e.g. when an LLM is helping the user login).

To further unify and reconcile across authentication mechanisms, would it be possible to create a declarative browser-mediated login flow that websites could use before users even get to the NASCAR flag UI?

# Goals

There are many conflicting goals that we are navigating, but here are a few that we have found useful to constrain the solution space:

- Must cover the most common login mechanisms, specifically passwords/passkeys and federation
- Must allow website authors to provide declarative login semantics to browsers (to allow use outside of the content area, e.g. the url bar or agentic browsing)
- Must be able to be retrofited into existing websites (e.g. support feature detection)

# Proposal

The proposal is to introduce a `<login>` element (along with a `<credential>` element) that can declaratively describe an imperative Credential Management API call, along the lines of `<geolocation>`.

We are still trying to figure out what are the right semantics, but one intuition is that `<login>` could work like `<a>` elements: an inline element that renders its inner contents and performs an action when users click on it.

The intention is to replace the typical "login" links that show up on the top right corner of websites with the following:

```html
<login onselect="login()">
  <credential type="publickey" 
      challenge="1234"
      rpId="example.com"
      userVerification="preferred"
      timeout="60000">
  </credential>
  <credential type="federated"
    clientId="1234"
    configURL="https://idp1.example/config">
  </credential>
  <credential type="federated"
     clientId="5678"
     configURL="https://idp2.example/config">
  </credential>
  <a href="login.html">login</a>
</login>

```

When clicked, a modal dialog is shown, with a browser mediated unified account chooser that contains all of the options specified by the developer.

# Sequencing

## Phase 1

While a unified modal dialog for login seems like a great end state, it doesn't quite seem like something that is quite within reach today: [`immediate mediation`](https://github.com/w3ctag/design-reviews/issues/1092) is still unresolved and the developer demand for a unified account chooser is still in its infancy.

One way that occurred to us that we could take a smaller and concrete step towards the right direction is to wrap the individual options in the NASCAR flag, rather than the top right corner login link.

So, for example, as opposed to replacing the "login" top right corner links, we'd replace the individual passkeys buttons in the NASCAR flag:

```html
<login onselect="login()">
  <credential type="publickey" 
      challenge="1234"
      rpId="example.com"
      userVerification="preferred"
      timeout="60000">
    <a onclick="navigator.credentials.get({publicKey: ...})">Sign-in with a Passkey</a>
  </credential>  
</login>
```

And the following for social login buttons:

```html
<login onselect="login()">
  <credential type="federated"
    clientId="1234"
    configURL="https://idp1.example/config">
    <a onclick="navigator.credentials.get({identity: ...})">Sign-in with IdP</a>
  </credential>  
</login>
```

This would allow us to introduce `<login>` for each of these individual mechanisms independently while working our way towards a unified UI for all of them, and ultimately replace the top-right-corner login links.

## Phase 2

Once we have each individual credential type working independently, it becomes a lot easier to unify them into a single UI. We'd expect that much of that work would result in the development of [`immediate mediation`](https://github.com/w3ctag/design-reviews/issues/1092) and would allow us to arrive at the desired end state.

# Open Questions

- Can/should developers be able to control whether the `<login>` element is a "semantics only" element (such as `<search>`) so that it can be deployed exclusively in agentic browsers (but not affect regular users?)? If so, how?

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

