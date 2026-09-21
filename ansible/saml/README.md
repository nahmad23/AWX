# AWX SAML SSO configuration

Reference values for enabling SAML SSO on AWX
(`https://uxus1sitbawx02.unitedlex.global`).

These are pasted into **Settings → Authentication → SAML** in the AWX UI
(or applied via the AWX settings API). AWX stores them in its own database —
keeping them here is for reference and reproducibility.

## Files here (safe to commit — non-secret)

| File | AWX field |
|------|-----------|
| `sp_org_info.json` | SAML Service Provider Organization Info |
| `sp_contact.json`  | SAML Service Provider Technical Contact **and** Support Contact |
| `saml_sp.crt` (add yourself) | SAML Service Provider Public Certificate — a **public** cert, safe to commit |

## Do NOT commit

- **`saml_sp.key`** (the SP private key) — secret. Keep it in `ansible-vault`
  (like `awx_admin_password`), never in git. It's covered by `.gitignore`
  (`*.key`).

## SP endpoints (for your IdP)

- SP Entity ID:  `https://uxus1sitbawx02.unitedlex.global`
- SP metadata:   `https://uxus1sitbawx02.unitedlex.global/sso/metadata/saml/`
- ACS / Reply:   `https://uxus1sitbawx02.unitedlex.global/sso/complete/saml/`
- Login start:   `https://uxus1sitbawx02.unitedlex.global/sso/login/saml/?idp=<IdP_NAME>`

## From the IdP (to finish, in "Enabled Identity Providers")

Per IdP you provide: `entity_id`, `url` (IdP SSO URL), `x509cert` (IdP signing
cert), and attribute mappings (`attr_user_permanent_id`, `attr_email`,
`attr_first_name`, `attr_last_name`, `attr_username`).
