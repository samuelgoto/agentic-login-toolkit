Author: @samuelgoto
Date: Jan 12, 2026
Status: early draft

# Agentic Login Toolkit

## Problem Statement

In LLM-powered Agentic browsers, many user journeys involve logging in to websites.  As with most of the LLM-powered actuation, the LLM has a baseline understanding using statistical models that allows it to click on links and fill forms to assist the user through the process.

However, as much as user agents are and should develop as many heuristics as possible to retrofit into the existing content on the Web, heuristics are, by design, unreliable and are expected to have lower precision and quality than structured content that is opted-into by website owners.

Not every website developer will have the incentives (expertise or demand) to annotate their content to be accessed by assistive browsers, but for those that do, what’s the best way that they can annotate their page to make user agents (e.g. browsers and search engines) better aware of their login flows?

This is a set of proposals to allow websites to expose to agentic browsers how their login flows are structured, so that they can be used as structured tools, rather than unstructured actuation. 

## Goals

There are probably a few conflicting goals, but here are a few that occur to me:

- Ergonomics matter: it should be easy for developers (and framework authors) to retrofit this annotation into their existing pages. The benefits for developers might be low, so the easier we can make the better (e.g. ARIA struggles to get adoption because the benefit/cost ratio is low).
- Tailwinds: it helps if the design interoperates with other assistive agents, like search engines or chatbots, so that we can increase the incentives for developers to adopt and maintain the annotation.
- Privacy: ideally, the user agent should make its “agentic mode” and an “non-agentic mode” indistinguishable from each other for the website to the extent that it is possible

## Proposal

There are many serializations and ontologies under consideration, but just to fix on something, let’s start with a proposal to extend schema.org/LoginAction and, if necessary, microdata:

https://github.com/schemaorg/schemaorg/issues/4507#issuecomment-3740859356

The proposal is to support all sorts of “login” forms, such as:

- Finding the login page in the first place
- Usernames and Password/Passkeys Forms
- Email/SMS OTP Forms
- Social Login Buttons
- Password Reset Forms

### Login page discovery

When agents get to a website and want to login the user to it, they need to first find the login page in the first place. This often involves a series of heuristics and computer vision, but the developer can help the agent find it with the following convention:

```html
<link rel="login" href="login.html">
```

(see alternatives considered for things like a .well-known file and other ways we can accomplish this)

### Federation

LoginAction has a property called `federated` which describes what the FedCM request would be.

The semantics of this annotation is that an assisted browser would be able to assume that “this is social login button”, and that “clicking on it” (which makes it indistinguishable from) would lead to an equivalent FedCM request.

```html
<div itemscope itemtype="https://schema.org/LoginAction">
  <data itemprop="federation" 
    value="client_id=\"1234\", config_url=\"https://idp.example/fedcm.json\"" />
  <data itemprop="target" value="js_callback">
  <button>Sign-in with X</button>
</div>
```

## Alternatives Under Consideration

There are two dimensions to be considered here with various options in each one: (a) how to encode semantics and (b) which ontology to use.

Here are a few ones that I’m aware of:

### Serialization
- microformats
- RDFa
- JSON-LD
- Use the `<data>` element
- Use the `data-*` attribute
- The `autocomplete` attribute
-- Something entirely new?

### Semantics

- Activity Streams
- Should we use https://schema.org/RegisterAction instead?
- The `autocomplete` taxonomy
- Something entirely new?
- Something new?
- Invent a new attribute
- WebMCP: https://github.com/webmachinelearning/webmcp/issues/22 


### Mediation: `conditional`

In this variation, we use the `mediation="conditional"` parameter to let the agent operate in the unresolved promise.

```javascript
const {token} = await navigator.credentials.get({
  mediation: "conditional",
  identity: { /** ... params ... */ }
});
```

Problem: 

- Reveals to the website that we are operating under an assistant
- https://github.com/schemaorg/schemaorg/issues/4507#issuecomment-3740859356

### JSON-LD

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "LoginAction",
  "federation": {
    "providers": [{
      "@type": "FederatedLoginProvider",
      "configURL": "https://idp.example/config.json",
      "clientId": "1234",
      "nonce": "4567",
      "fields": ["email", "name", "picture"],
     }]
  },
}
</script>
```

The result of the action gets returned as a DOM event:

```javascript
document.addEventListener("action", ({type, token}) => login(token));
```

## Alternatives Considered

### Overload WWW-Authenticate

In this variation we’d support a declarative request made via HTTP headers, like WWW-Authenticate or introduce a few one:

```
Federated-Authentication: 
  client_id=\"1234\", config_url=\"https://idp.example/fedcm.json\""
```

Cons:

- Requires RPs to redeploy their servers
- WWW-Authenticate is blocking (and because of that, we think, poorly adopted)

### <meta>

In this variation we’d use the <meta> tag disassociated with the element to be clicked.

```html
<meta http-equiv="Federated-Authentication" 
  content="client_id=\"1234\", config_url=\"https://idp.example/fedcm.json\""
>
<script>
document.addEventListener("login", ({token}) => login(token));
</script>
```

Problem: 

- Reveals to the website that we are operating under an assistant

# Open Questions

Should this support also Passwords/Passkeys too?

### Usernames/Passwords (or Passkeys)

```html
<form itemscope itemtype="https://schema.org/LoginAction">
  <input type="email" autocomplete="username">
  <input type="password" autocomplete="webauthn">
  <button itemprop="target" type="submit">Login</button>
</div>
```

### OTPs

```html
<form itemscope itemtype="https://schema.org/LoginAction">
  <input itemprop="otp" autocomplete="one-time-code">
  <button>Login</button>
</div>
```

### Password Reset Forms

```html
<form itemscope itemtype="https://schema.org/ResetPasswordAction">
  <input itemprop="username" type="email">
  <button itemprop="target" type="submit">Reset password</button>
</div>
```

### Extensibility

```html
<form itemscope itemtype="https://schema.org/Action">
  <data itemprop="name" value="My custom form that does a bunch of stuff">
  <data itemprop="description" 
    value="This form creates an account for the user and sends them a gift from Macys">
  <input itemprop="macys" name="macyscard">
</form>
```
