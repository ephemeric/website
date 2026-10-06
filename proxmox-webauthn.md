### [[Simplicity]](simplicity.md) [[Complexity]](complexity.md) [[Neomania]](neomania.md) [[Networking]](networking.md) [[Security]](security.md) [[Miscellaneous]](miscellaneous.md)

# Proxmox WebAuthn: YubiKey enrolment failure on macOS

**Observed:** 6 October 2026\
**Purpose:** Concise reproduction and diagnostic summary\
**Investigation and write-up:** Simone (ChatGPT, OpenAI), Robert and Bobby (dog)

## Summary

A YubiKey that works on other WebAuthn/passkey sites could not be
enrolled as a Proxmox VE WebAuthn second factor from macOS.

-   **Safari on macOS:** detected the YubiKey and prompted for its FIDO2
    PIN, but reported the known-correct PIN as incorrect.
-   **Firefox on macOS:** stalled while waiting for/touching the
    security key.
-   **macOS Keychain WebAuthn:** worked with the same Proxmox
    configuration.
-   **Debian 13 GNOME + Firefox:** successfully enrolled the same
    YubiKey.
-   Once enrolled from Debian, the YubiKey authenticated successfully
    from both Debian Firefox and macOS Safari.

The evidence therefore isolates the observed failure to **credential
creation/enrolment of the external YubiKey through the tested macOS
browser path**, rather than the YubiKey, its PIN, the Proxmox WebAuthn
configuration, or subsequent authentication.

## Environment

  Component               Observed configuration
  ----------------------- ----------------------------------
  Service                 Proxmox VE WebAuthn / TFA
  RP ID                   `pve.ephemeric.lan`
  Origin                  `https://pve.ephemeric.lan:8006`
  Authenticator           YubiKey FIDO2 / WebAuthn
  macOS browsers tested   Safari and Firefox
  Control client          Debian 13 GNOME + Firefox

Proxmox WebAuthn configuration:

``` text
webauthn: id=pve.ephemeric.lan,origin=https\://pve.ephemeric.lan:8006,rp=pve.ephemeric.lan
```

## Observed behaviour

  ---------------------------------------------------------------------
  Test                               Result
  ---------------------------------- ----------------------------------
  macOS Safari --- enrol YubiKey     **FAIL** --- prompts for PIN, then
                                     reports correct PIN as incorrect

  macOS Firefox --- enrol YubiKey    **FAIL** --- hangs waiting
                                     for/touching security key

  macOS Keychain WebAuthn            **PASS**

  Direct CTAP2 PIN verification      **PASS** --- PIN verified; 8
                                     attempts remain

  Debian 13 Firefox --- enrol        **PASS** --- PIN prompt, touch,
  YubiKey                            credential created

  Debian 13 Firefox --- authenticate **PASS** --- touch only; no PIN
                                     prompt

  macOS Safari --- authenticate      **PASS** --- simply touch YubiKey;
  enrolled YubiKey                   no PIN and no need to select
                                     "Security key"
  ---------------------------------------------------------------------

## Key diagnostic evidence

The YubiKey PIN was independently verified using YubiKey Manager:

``` text
$ ykman fido access verify-pin
Enter your PIN:
PIN verified.

$ ykman fido info
AAGUID:             ee882879-721c-4913-9775-3dfcce97072a
PIN:                8 attempt(s) remaining
Minimum PIN length: 4
```

This establishes that:

1. The FIDO2 PIN was correct.
2. The YubiKey FIDO2 application was functioning.
3. macOS could communicate with the key over CTAP2.
4. The failed browser enrolment did not consume a PIN retry.

Safari's "incorrect PIN" message was therefore misleading in this test.

## Control experiment

The decisive test was to change the client platform/browser used for
**registration**, while keeping the Proxmox installation, relying party
and YubiKey unchanged.

The same YubiKey was enrolled from **Debian 13 GNOME + Firefox**.

During enrolment:

``` text
Debian Firefox
    -> FIDO2 PIN prompt
    -> touch YubiKey
    -> credential successfully created
```

Subsequent authentication from Debian Firefox worked by touching the
YubiKey, with no PIN prompt.

The newly created credential was then tested from **macOS Safari**.
Authentication also succeeded.

Importantly, Safari did **not** require the user to select the "Security
key" radio option. While Safari's WebAuthn authenticator-selection
dialog was displayed, simply touching the inserted YubiKey completed
authentication.

## Screenshot

The screenshot below shows Safari's WebAuthn prompt immediately before
successful authentication. The **Security key** radio option is visibly
not selected; touching the already-enrolled YubiKey was sufficient.

![Safari WebAuthn prompt --- touching the enrolled YubiKey authenticates
without selecting Security
key](safari-webauthn-prompt.png)

## Result matrix

``` text
YubiKey FIDO2 application                     PASS
YubiKey FIDO2 PIN                             PASS
macOS -> YubiKey CTAP2 PIN verification       PASS
Proxmox WebAuthn RP/origin                    PASS
macOS Keychain -> Proxmox enrolment           PASS

macOS Safari -> YubiKey enrolment             FAIL
macOS Firefox -> YubiKey enrolment            FAIL

Debian Firefox -> YubiKey enrolment           PASS
Debian Firefox -> enrolled YubiKey login      PASS
macOS Safari -> enrolled YubiKey login        PASS
```

## Conclusion

The tests reproduce a failure when **creating/enrolling an external
YubiKey WebAuthn credential through the tested macOS browser paths**.

The failure is not explained by:

- an incorrect FIDO2 PIN
- exhausted PIN retries
- a defective YubiKey
- an invalid Proxmox RP ID or origin
- a general Proxmox WebAuthn failure
- incompatibility of the resulting YubiKey credential with macOS Safari

Once the YubiKey credential was provisioned from Debian Firefox, macOS
Safari could use it normally for authentication.

### Practical workaround

**Enrol the YubiKey against Proxmox from Debian 13 Firefox, then use the
enrolled YubiKey normally from macOS Safari.**

## Scope

This document records the observed reproduction and isolation. It does
**not** establish which underlying macOS, Safari, Firefox, WebAuthn or
CTAP2 implementation component is responsible for the enrolment failure.

### [[Please report any broken links via GitHub. Suggestions welcome. Polemics unwelcome.]](https://github.com/ephemeric/website)
