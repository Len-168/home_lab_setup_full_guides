# Protect Your Server with Cloudflare Zero Trust (Access)

Based on Cloudflare's official guides: [Add a self-hosted application](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/self-hosted-public-app/), [Access policies](https://developers.cloudflare.com/cloudflare-one/policies/access/) and [One-time PIN login](https://developers.cloudflare.com/cloudflare-one/identity/one-time-pin/).

> **Do this before you publish any hostname (like SSH) through the tunnel.**
> If you publish first, the hostname is reachable by anyone on the internet until Access is in place.

## What is Zero Trust, and why use it?

**Zero Trust** means "never trust, always verify". No one is trusted just because they know the address of your server. Every request must prove **who** it is before it gets through.

**Cloudflare Access** is the part of Zero Trust that does this. It puts a login page **in front of** your server, at Cloudflare's edge:

```text
Without Access                          With Access

Internet                                Internet
   ↓                                       ↓
Tunnel                                  Cloudflare Access  ← "Who are you?" (email / login + MFA)
   ↓                                       ↓  only allowed users pass
SSH server  ← anyone can try            Tunnel
              to log in                    ↓
                                        SSH server  ← still asks for your SSH key
```

What you get:

- **No open ports.** The tunnel only connects outward, so port scanners find nothing.
- **Two locks, not one.** An attacker needs your Cloudflare login **and** your SSH private key.
- **Only your email is allowed.** Everyone else is stopped at Cloudflare and never reaches your server.
- **A log of every login.** Cloudflare records who logged in and when (Zero Trust → Logs → Access).

## Before you start

- The tunnel is created and **Healthy**: [2-setup-tunnel.md](2-setup-tunnel.md)
- You know which hostname you will use, for example `ssh.YOUR-DOMAIN.com`

## Step 1: Choose how you log in (identity provider)

An **identity provider** is the service that checks who you are. Pick one:

| Option | How it works | Good for |
|---|---|---|
| **Cloudflare account** (default for new organizations) | Log in with your Cloudflare account | Best long-term, if your Cloudflare account has **MFA turned on** |
| **One-time PIN (email OTP)** | Cloudflare emails you a code | Simple, good for learning and testing |
| **Google / GitHub / etc.** | Log in with an existing account | If you already use one with MFA |

Whichever you pick, **turn on MFA** (two-factor login) for that account. Your server is only as safe as that login.

### Optional: Turn on email One-time PIN

New Zero Trust organizations no longer get One-time PIN set up automatically. To use the "send a code to my email" login, add it yourself:

```text
Zero Trust
   ↓
Integrations
   ↓
Identity providers
   ↓
Add new identity provider
   ↓
One-time PIN
```

The codes can only be used once and expire 10 minutes after you request them.

## Step 2: Create the Access application

The **application** tells Cloudflare which hostname to protect.

```text
Zero Trust
   ↓
Access controls
   ↓
Applications
   ↓
Create new application
   ↓
Self-hosted and private
```

Fill in:

```text
Application name:   Home Lab SSH
Public hostname:    ssh  .  YOUR-DOMAIN.com
                    ↑ subdomain   ↑ domain
Session duration:   24 hours (or shorter)
```

**Session duration** is how long you stay logged in before Cloudflare asks again. Shorter is safer.

## Step 3: Create the Allow policy

The **policy** decides who gets in. Add a policy to the application:

```text
Policy name:   Home Lab Admin
Action:        Allow
Include:       Emails
Value:         your-email@example.com
```

Then **Save policy** and **Save application**.

How Cloudflare reads this:

- **Include** = who the rule is for. Here: only your exact email address.
- Anyone who doesn't match an **Allow** policy is **blocked automatically**. You don't need a separate "deny everyone" rule.

> **Don't** use `Include: Everyone` or allow a whole email domain like `@gmail.com`. That would let any Gmail user in.

### Optional: Make the policy stricter

Under **Require** in the same policy you can add more conditions. A user must match **all** of them:

| Require | Example | Effect |
|---|---|---|
| Country | Your country | Logins from other countries are blocked |
| Login methods | One-time PIN / your provider | Only that login method is accepted |
| Authentication method | `mfa` | Only logins that used MFA are accepted (works with providers that report MFA) |

## Step 4: Check the application is in place

In **Access controls → Applications** you should see **Home Lab SSH** with your hostname.

Now it is safe to publish the hostname through the tunnel: [4-publish-ssh-tunnel.md](../3%20-%20ssh/4-publish-ssh-tunnel.md).

## How a login works after setup

```text
You run: ssh dev-ssh
     ↓
cloudflared opens the Cloudflare login page in your browser
     ↓
You log in (email + OTP, or Cloudflare account + MFA)
     ↓
Cloudflare checks your policy  →  not allowed? blocked here
     ↓
Access passes, traffic goes through the tunnel
     ↓
SSH server checks your private key  →  no key? blocked here
     ↓
Ubuntu shell
```

You only log in to Cloudflare once per session duration. After that, `ssh` connects straight away.

## Security checklist

- [ ] The Access application was created **before** the hostname was published.
- [ ] The policy allows **only your email**, not `Everyone` or a whole domain.
- [ ] Your login account (Cloudflare, Google, etc.) has **MFA** turned on.
- [ ] SSH on the server allows **key login only**, with password login turned off (see [3-setup-ssh-key.md](../3%20-%20ssh/3-setup-ssh-key.md)).
- [ ] The tunnel token is not saved anywhere public.
- [ ] Every now and then, check **Zero Trust → Logs → Access** for logins you don't recognise.

## Troubleshooting

| Problem | Fix |
|---|---|
| No OTP email arrives | Check spam. Make sure One-time PIN is added under **Identity providers** and selected in the application's login methods. |
| "You don't have access" after logging in | The email in the policy doesn't match the one you logged in with. Check for typos. |
| SSH connects with no Cloudflare login | The hostname isn't covered by an application. Check the hostname in **Applications** exactly matches the one in the tunnel. |
| Login page keeps coming back | Your session expired, or your browser blocks cookies for `*.cloudflareaccess.com`. |
