# Trusteed AgenticTools for PrestaShop ≤ 2.3.0 — Unauthenticated fail-open bypass of all checkout enforcement rules

| | |
|---|---|
| **Product** | Trusteed AgenticTools (`agentic-commerce-prestashop`) — PrestaShop module |
| **Vendor** | Trusteed (trusteed.xyz) |
| **Affected versions** | ≤ 2.3.0 |
| **Fixed version** | 2.3.1 (released 2026-09-14) |
| **Vendor advisory** | [GHSA-2j2x-5q52-g48m](https://github.com/Trusteedxyz/agentic-commerce-prestashop/security/advisories/GHSA-2j2x-5q52-g48m) |
| **CVE** | Pending (requested via the vendor's GitHub advisory) |
| **Severity** | High — CVSS 3.1 7.5 (`AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N`) |
| **CWE** | CWE-636 (Not Failing Securely / fail-open), CWE-755 |
| **Researcher** | Salúa Es-sair — [LinkedIn](https://www.linkedin.com/in/sal%C3%BAa-es-sair/) |

## Summary

Trusteed enforces merchant checkout policy (the "CEL" — Checkout Enforcement Layer): country restrictions, business hours, maximum order amount, price-tamper detection, replay protection and a human-review gate for suspicious orders. All of this runs inside a single `try/catch` in a `PaymentModule::validateOrder()` override that blocks on the module's own `PrestaShopException` but **fails open** (logs and lets the order proceed) on any other exception.

An unauthenticated checkout request carrying a malformed agent token makes the Ed25519 signature check throw a `SodiumException` (wrong signature length), which falls into the fail-open branch. Every CEL rule is skipped for that order.

## Root cause

- `src/Enforcement/TokenVerifier.php` passes the attacker-supplied, base64url-decoded signature directly to `sodium_crypto_sign_verify_detached()` without checking that it is exactly `SODIUM_CRYPTO_SIGN_BYTES` (64) bytes long. For any other length the function **throws** instead of returning `false`.
- `override/classes/PaymentModule.php::validateOrder()` treats every non-`PrestaShopException` error as an infrastructure failure and continues with order creation.

## Impact

Any checkout (not only AI-agent ones) can bypass all merchant enforcement rules: geo/business-hours blocks, maximum order amount, price-tamper and replay protection, and the human-in-the-loop review gate. This defeats the module's core purpose on every store running it.

## Fix

Fixed in **2.3.1**: the signature length is validated before verification and the fail-open path no longer covers errors raised while parsing attacker-controlled input. Update to 2.3.1 or later.

## Timeline

- 2026-09-11 — Reported privately to the vendor
- 2026-09-13 — Vendor confirmed the issue
- 2026-09-14 — Fix released in 2.3.1 and advisory GHSA-2j2x-5q52-g48m published
- 2026-10-09 — CVE assignment requested from the vendor via GitHub; this write-up published

Thanks to the Trusteed team for a fast, professional response.
