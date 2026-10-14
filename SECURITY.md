# Security Policy

## Project Status

SecurityMonitor is a research prototype developed as part of a Bachelor's thesis at Turku University of Applied Sciences in 2025.

This repository documents a security monitoring approach for federated learning environments. It is not a production-ready security product, and no formal security support or vulnerability remediation schedule is provided.

## Supported Versions

There are currently no officially supported production releases.

| Version | Supported |
|---|---|
| Research prototype (2025) | No formal security maintenance |

## Reporting a Vulnerability

If you discover a potential security vulnerability in this project, please report it privately rather than opening a public GitHub issue.

Where available, use GitHub's **Report a vulnerability** feature under the repository's **Security** tab.

Include the following information when possible:

- A description of the vulnerability and its potential impact.
- The affected component or configuration.
- Steps to reproduce the issue.
- Relevant logs or proof-of-concept details, with sensitive information removed.
- Suggested mitigations, if known.

Please do not include credentials, personal information, confidential research data, or other sensitive material in public reports.

## Scope and Limitations

SecurityMonitor was designed to investigate tampering and repudiation risks through HMAC-based audit logging, independent filesystem monitoring, and event correlation.

The prototype does not guarantee prevention of model poisoning, complete detection coverage, or protection against attackers with privileged access to the underlying host.

Security findings should be assessed in the context of the original research environment.

## Responsible Disclosure

Please avoid publicly disclosing potentially exploitable vulnerabilities before the relevant maintainers have had a reasonable opportunity to assess them.

For vulnerabilities affecting third-party components or the original SA-FLaaS platform, contact the appropriate project maintainers or vendors through their established security reporting channels.

## Disclaimer

This project is provided for educational, research, and portfolio purposes. Its documentation and examples should not be treated as production security guidance without independent review and appropriate hardening.
