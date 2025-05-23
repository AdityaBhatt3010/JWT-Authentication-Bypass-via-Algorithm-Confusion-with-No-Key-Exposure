# 🔓 JWT Authentication Bypass via Algorithm Confusion (No Key Exposure)

![JWT_Final_Cover](https://github.com/user-attachments/assets/b506deee-9fe9-4270-af24-3e61bed517b3) <br/>

---

## 🧠 Introduction: JWT, But Not Quite Secure…

JSON Web Tokens (JWTs) are everywhere — they’re fast, stateless, and perfect for modern web authentication. But with great power comes great misconfigurations. In this write-up, we’ll dive into a juicy `JWT algorithm confusion` vulnerability, where we bypass authentication by exploiting a subtle flaw — **without access to any private key**.

This was part of PortSwigger's **Expert-level lab** and demonstrates how **poor JWT implementation decisions** can be catastrophic, even when using secure RSA public-private key pairs.

---

## ⚠️ What is JWT Algorithm Confusion?

JWT supports multiple signing algorithms, most notably **HS256 (HMAC using SHA-256)** and **RS256 (RSA Signature using SHA-256)**. Algorithm confusion happens when a server that is configured to verify **RS256 tokens** fails to enforce that, and instead **accepts HS256 tokens** where the attacker provides the public key as the secret.

When the verification step treats the **public key** (intended for asymmetric RSA validation) as a **secret for symmetric HMAC**, it enables full control over token signing — essentially letting attackers forge any identity.

---

## 🔑 Quick JWT Header Rundown (for the curious):

* `alg`: Algorithm used to sign the token (e.g., RS256, HS256).
* `kid`: Key ID — can be used to load different keys (dangerous when dynamic or unsanitized).
* `jku`: JSON Web Key Set URL — can be abused if the server fetches signing keys from attacker-controlled URLs.
* `jwk`: JSON Web Key — embeds a public key in the token header (can also be abused under misconfiguration).

💡 *This lab doesn’t rely on `kid`, `jku`, or `jwk`, but they are common vectors in real-world JWT attacks and bounty reports.*

---

## 🔬 Vulnerability Summary

**Type**: JWT Authentication Bypass
**Weakness**: Algorithm confusion (RS256 → HS256)
**Impact**: Admin access and account deletion
**CVE-like Relevance**: Improper Authorization Check via JWT Algorithm Trust
**Bounty Potential**: High — this would be a P1 (Critical) in real-world apps

---

## 🔍 Exploit Walkthrough (Step-by-Step)

Let’s recreate the lab and walk through the exact exploit chain.

---

### 🔧 Background: Install JWT Editor Extension (Burp BApp Store)

![JWTEditor](https://github.com/user-attachments/assets/8739da36-27cc-4d7b-8e4c-ef97799e0996) <br/>

---

### ✅ 1. Access the Lab

Visit:
`https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion-with-no-exposed-key`
Login with: `wiener:peter`

![1](https://github.com/user-attachments/assets/1ac86966-878c-4a82-8941-89bda4fc88fb) <br/>

---

### 🧪 2. Try Accessing /admin

Modify the `/my-account` request to `/admin`.
You see:

> "Admin interface only available if logged in as an administrator."

![2](https://github.com/user-attachments/assets/3277ad10-acce-4f3a-805c-9be67236b6b7) <br/>

---

### 🔄 3. Relogin and Collect JWTs

* Logout, then log in again.
* Capture both sessions' JWTs from the `session` cookie.
* Save them for public key derivation.

![3](https://github.com/user-attachments/assets/a2452688-af54-44e1-a8b5-4432eb132c26) <br/>

---

### 🔑 4. Run the Key Derivation Script

Use Docker’s `sig2n` tool to brute-force the public key:

```bash
docker run --rm -it portswigger/sig2n <JWT1> <JWT2>
```

This outputs:

* Possible `n` values (modulus)
* Base64-encoded **X.509 public key**
* Tampered JWTs signed using that key

![4](https://github.com/user-attachments/assets/c8ce1c5e-4526-4ec5-b71d-7527f6fdecb8) <br/>

---

### 🎯 5. Identify the Valid JWT

* Paste each tampered JWT into the `session` cookie.
* Send a request to `/my-account`.

✅ If you get a `200 OK` → This is the correct key.
🚫 If you get a `302 Redirect to /login` → Try the next.

![5](https://github.com/user-attachments/assets/174ff03b-54f2-4cb4-988e-e66eb4eb74ed) <br/>

---

### 🧬 6. Create a Symmetric Key (JWT Editor)

* Copy the **Base64 X.509 key** from the valid tampered JWT.
* In JWT Editor > Keys tab → New Symmetric Key.
* Set the `k` value to the Base64 public key.

![6](https://github.com/user-attachments/assets/b798b5ee-3123-42a1-91c9-6a2741b098b5) <br/>

---

### 🛠️ 7. Modify and Sign the Token

* In Burp Repeater > JSON Web Token tab:

  * Change `sub` claim to `administrator`
  * Set `alg` to `HS256`
  * Click **Sign** → Choose the newly created key
  * Ensure “Don’t modify header” is checked

![7](https://github.com/user-attachments/assets/c9c45e51-2fed-41d2-9df4-221da5ac190f) <br/>

---

### 🔓 8. Send the Request to /admin

And *Voila!* 🪄
You now have **admin access**!

![8](https://github.com/user-attachments/assets/239133cd-3d4c-4059-a303-d9e2a1d9a24b) <br/>

---

### 💣 9. Delete User Carlos

Send a request to:
`/admin/delete?username=carlos`

![9](https://github.com/user-attachments/assets/68356b42-ed70-4bf0-a404-87cbe3b7a199) <br/>

---

### 🎉 10. Confirm the Lab Is Solved

Right-click → Show in browser → Paste the response URL.
Lab solved ✔️

![10](https://github.com/user-attachments/assets/5e7e6113-eb8f-436b-b230-6159893713e0) <br/>

---

## 💡 Key Takeaways

* Never trust the `alg` parameter from untrusted sources.
* Always **enforce algorithm validation** on the server side.
* Public keys are… public! Don’t use them as secrets.
* Even when using strong cryptography (RSA), poor implementation = **total compromise**.

---

## 🧑‍💻 Final Thoughts

This lab beautifully demonstrates how a secure mechanism like JWT can be weaponized if developers allow `alg: HS256` without proper verification. Attackers don't need your private key — just a flaw in trust.

If you're building or assessing apps using JWT, **enforce fixed algorithms**, avoid dynamic keys via `kid`/`jku` unless fully controlled, and never treat **public keys as secrets**.

---

## 🧡 Happy Hacking!

Stay curious, stay ethical — and always double-check your crypto assumptions.

Cheers,
- **Aditya Bhatt**

---
