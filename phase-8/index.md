---
layout: default
title: "Phase 8: Security at Scale"
nav_order: 9
has_children: true
permalink: /phase-8/
---

# Phase 8: Security at Scale
{: .no_toc }

**Duration:** Weeks 33–35
{: .label .label-yellow }

Security is not a feature added at the end — it is an architectural constraint applied from the start. This phase covers how large-scale systems implement identity and authorization (Zero Trust, RBAC/ABAC, Keycloak, SAML), harden against application-layer attacks (OWASP Top 10, secrets management, encryption), and meet regulatory compliance requirements (GDPR, HIPAA, PCI-DSS). These topics appear in Staff Engineer interviews because they require understanding of cross-cutting system concerns that shape every layer of the stack.
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Authentication & Authorization](authn-authz/) | Zero Trust, RBAC vs ABAC, Keycloak, LDAP, SAML 2.0 |
| 2 | [Application Security](application-security/) | OWASP Top 10, SQL Injection, XSS/CSRF, Secrets Management, TLS 1.3, SAST/DAST |
| 3 | [Compliance & Data Privacy](compliance/) | GDPR, HIPAA, PCI-DSS, Data Masking, PII Handling |

---

## Why Phase 8 Matters

Security topics separate pragmatic senior engineers from theoretical ones:

- A new microservice needs to call an internal API — how do you authenticate service-to-service without hardcoding credentials? (**mTLS + Vault dynamic secrets**)
- A user's account is breached but the attacker is already inside the corporate VPN — how is your perimeter-based security model a liability? (**Zero Trust: authenticate every request, regardless of network location**)
- An engineer accidentally commits an API key to GitHub — how does your system mitigate and rotate the secret without a manual process? (**Vault dynamic secrets + automated rotation**)
- A GDPR deletion request arrives — how does your event-sourced system "erase" data from an immutable log? (**Cryptographic erasure + data segregation**)
- A penetration test found an SSRF vulnerability — what exactly is it and how does Spring Boot mitigate it? (**Untrusted URL parameters → allowlist validation**)
- Your CI/CD scanned 800 dependencies — which CVEs are critical vs acceptable risk? (**CVSS scoring, OWASP Dependency-Check**)

These are decisions that block production deployments and shape system architecture.

---

## Phase 8 Checklist

- [ ] Describe Zero Trust Architecture — what is "never trust, always verify" and how does it differ from perimeter security?
- [ ] Explain the difference between authentication (AuthN) and authorization (AuthZ)
- [ ] Compare RBAC and ABAC — give a scenario where ABAC is required that RBAC cannot handle
- [ ] Describe how Keycloak integrates with Spring Boot as an OAuth2 resource server
- [ ] Explain how SAML 2.0 SP-initiated SSO works — which parties are involved and what assertions flow
- [ ] Describe how Spring Security LDAP bind authentication works
- [ ] List all 10 OWASP Top 10 (2021) vulnerabilities and the primary mitigation for each
- [ ] Show the difference between a vulnerable SQL query and a parameterized equivalent in Java
- [ ] Explain how Spring Security's CSRF protection works and when to disable it
- [ ] Describe how HashiCorp Vault dynamic secrets work for database credentials
- [ ] Explain TLS 1.3 improvements over TLS 1.2 — handshake rounds, perfect forward secrecy
- [ ] Describe envelope encryption with KMS — how the data key and master key relate
- [ ] Explain what SAST and DAST are — give one tool for each and what they catch
- [ ] Define GDPR's "right to erasure" — how does it apply to event-sourced systems?
- [ ] Describe PII handling in logs — what should never appear and how to mask it
- [ ] Explain tokenization vs encryption for PCI-DSS cardholder data
- [ ] Describe cryptographic erasure as a GDPR deletion strategy

---

## References

- *The Web Application Hacker's Handbook* — Stuttard & Pinto
- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/index.html)
- [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [NIST Zero Trust Architecture SP 800-207](https://doi.org/10.6028/NIST.SP.800-207)
- [GDPR Official Text](https://gdpr-info.eu/)
