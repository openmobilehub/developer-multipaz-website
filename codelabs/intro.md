---
sidebar_position: 3
---

# Intro


# Multipaz

This repository contains libraries and applications for working with real-world
identity. The initial focus for this work was mdoc/mDL according to [ISO/IEC 18013-5:2021](https://www.iso.org/standard/69084.html)
and related standards but the current scope also include other credential formats and
presentment protocols.

## Multipaz Libraries

The project provides libraries written in [Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform.html):

- `multipaz` provides the core building blocks it works on Android,
  iOS, and in server-side environments. 
- `multipaz-models` is a Kotlin Multiplatform library with stateful models
  with business logic intended to be used by graphical applications.
- `multipaz-compose` provides rich UI elements to be used in Compose
  applications.
- `multipaz-android-legacy` contains an older version of the APIs for
  applications not yet migrated to the newer libraries. 
- `multipaz-doctypes` contains known credential document types (for example
  ISO/IEC 18013-5:2021 mDL and EU PID) along with human-readable descriptions
  of claims / data elements, sample data, and sample requests.

A command-line tool `multipazctl` is also included which can be used to generate
ISO/IEC 18013-5:2021 IACA certificates among other things. Use
`./gradlew --quiet runMultipazCtl --args "help"` for documentation on supported
verbs and options.


## Utopia Wholesale App 

This codelab includes the Utopia Wholesale App, a demo application that showcases how to use the identity-credential project and its Multipaz APIs to implement a complete digital identity flow — including credential issuance, secure storage, presentation, and verification.

This guide is designed for developers who want to:
- Understand the end-to-end flow of verifiable credentials using the OpenWallet Foundation standards.
- Learn how to integrate the Multipaz APIs into their own mobile or cross-platform apps (Kotlin Multiplatform / Compose Multiplatform).
- Explore how identity data can be issued by a trusted source, stored securely on-device, and presented to verifiers in a privacy-preserving way.

By following this Utopia example, you’ll get a hands-on look at how to build real-world identity experiences using the identity-credential toolkit.
