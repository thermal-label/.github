# Security policy

## Supported versions

The **thermal-label** packages are pre-1.0. Security fixes are applied to the
latest published minor version of each package. Older versions are not patched —
upgrade to receive fixes.

| Package family | Supported |
|---|---|
| `@thermal-label/contracts` | latest minor |
| `@thermal-label/transport` | latest minor |
| `@thermal-label/brother-ql-*` | latest minor |
| `@thermal-label/labelmanager-*` | latest minor |
| `@thermal-label/labelwriter-*` | latest minor |
| `thermal-label-cli` | latest minor |

## Reporting a vulnerability

**Please do not report security issues through public GitHub issues.**

Use GitHub's [private security advisory] flow on the affected repository:

1. Open the **Security** tab of the repository where the issue lives (e.g.
   `thermal-label/transport`)
2. Click **Report a vulnerability**
3. Fill in a clear description, reproduction, and impact assessment

Maintainers will acknowledge receipt within **5 business days**, provide an
initial assessment within **14 days**, and aim to ship a fix within **30 days**
of confirmation. Coordinated disclosure is appreciated; we will credit the
reporter in the release notes unless they prefer to remain anonymous.

## What counts as a security issue

The thermal-label packages talk to **physical printers** over USB, TCP, WebUSB,
Web Bluetooth, and Web Serial. Issues that may warrant a private report:

- Code paths that allow a malicious printer or network peer to execute code on
  the host (e.g. unbounded buffer copies in transport layers)
- Crashes or hangs reachable via crafted printer responses (denial of service
  against a long-running daemon)
- WebUSB / WebBluetooth flows that escalate beyond the device the user paired
- Dependencies pinned to a version with a known critical CVE that affects our
  usage

Bugs that affect printing correctness, hardware compatibility, or developer
ergonomics are **not** security issues — file them as normal issues on the
relevant repository.

[private security advisory]: https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability
