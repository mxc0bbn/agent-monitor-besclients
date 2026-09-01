# Acknowledgments

Agent Monitor for BES Clients has been shaped by the testing, security review,
and feedback of community members willing to put pre-release builds through
their paces against real environments and report what they found.

## Testing and quality

- **[@jgstew](https://github.com/jgstew)** - Contributed the project's automated
  test suite, a comprehensive pytest suite of over 750 tests spanning the
  dashboard, health agent, and forwarder, together with continuous integration
  through GitHub Actions and pre-commit hooks. The suite surfaced nine latent
  bugs that were fixed in the 2.7.1 release, and it continues to catch
  regressions on every change.

## Security assessment

- **[@jeffschafer](https://github.com/jeffschafer)** - Carried out a thorough
  vulnerability assessment of the application. His review surfaced multiple
  security gaps, each of which was remediated in the hardening work that
  followed.

## Beta testing

- **[Pawan Kumar](https://github.com/HCL-PawanWorks)** - Beta tested pre-release builds against a live environment
  and surfaced installation issues that shaped the 2.7.2 release. His validation
  identified a certificate name mismatch that blocked agent enrollment under
  strict TLS verification, and the agent's inability to locate BigFix when the
  client or its server components were installed outside the default
  directories. Both became out-of-the-box installation fixes in that release.

- Thanks also to additional collaborators who tested pre-release builds against
  external endpoints and whose feedback shaped two design decisions: placing the
  Windows code-signing certificate under the customer's own control, and
  strengthening the cryptographic design to post-quantum algorithms in place of
  the classical scheme originally planned.

---

If you run a pre-release build and want to be listed here, open a discussion or
reach out through the repository's issue tracker.
