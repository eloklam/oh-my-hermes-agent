# Security Policy

## Reporting security issues

Do not report sensitive issues or accidentally committed secrets in public GitHub issues.

If you find a credential or token in this repository:

1. **Rotate the token immediately** if it is yours.
2. Contact the maintainer privately through GitHub (private vulnerability reporting if enabled).
3. Do not open a public issue with the secret visible.

## Expected state

This repository should **never** contain:

- API keys or access tokens
- Private keys or certificates
- Live profile configs with secrets
- Gateway logs or debug dumps containing auth material

If you find any of the above in the repo, it was committed by mistake and should be reported privately.
