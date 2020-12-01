# README

This notes are the outcome of an effort to understand how to quickly set up SATOSA proxies for
publishing OIDC RPs in Research and Education Identity Federations as well as to bridge other
authentication systems.

References:
* https://github.com/IdentityPython/SATOSA
* https://lists.sunet.se/pipermail/satosa-users/
* https://lists.sunet.se/pipermail/satosa-users/2019-February/000066.html
* https://lists.sunet.se/pipermail/satosa-dev/

This notes are based on SATOSA version 7.0.1 and they have been tested on a Debian Buster
dstribution. 

## Use cases

So far the following use cases have been tested and documented:
- OpenID Connect Relying Party in SAML 2.0 Identity Federations:
  [oidc_rp-to-saml.md](oidc_rp-to-saml.md)
- Google Authentication as a SAML 2.0 Identity Provider:
  [google_auth-to-saml.md](google_auth-to-saml.md)

