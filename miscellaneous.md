### [[Simplicity]](simplicity.md) [[Complexity]](complexity.md) [[Neomania]](neomania.md) [[Networking]](networking.md) [[Security]](security.md) [[Miscellaneous]](miscellaneous.md)

# The Python Cheese Shop

## How We Got Here

It began innocently enough with a reluctant conclusion:

> I probably need to learn Python.

Python keeps appearing everywhere relevant to modern Linux infrastructure:

- Ansible is largely Python.
- AI/ML tooling is overwhelmingly Python-facing.
- PyTorch, Hugging Face and much of the NVIDIA ecosystem expose Python interfaces.
- Python remains heavily requested in Linux engineering jobs.
- It is useful for the space between Bash scripts and larger compiled applications.

There was only one problem:

> **I hate Python.**

After Rust, Go, Perl, Raku and years of Bash, Python wasn't exactly calling to the heart.

Then we found the right project:

**Write custom Ansible modules.**

Instead of learning Python through pointless exercises, use it to solve real infrastructure problems. Design the Ansible interface, implement idempotence, support check mode, return structured state, and learn Python while building something genuinely useful.

And importantly:

> **Abstraction should compress expertise, not substitute for acquiring it.**

Simone can review, explain and challenge the code without becoming the person who understands it instead.

Eventually the milestone becomes:

> "No, Simone. That's wrong because…"

At that point, Python has done its job.

---

# Enter the Rust Butler

For a portable Python installation, we discovered `uv`.

And then discovered the delicious irony:

**The tool making modern Python environments considerably less painful is written in Rust.**

Thus the **Rust Butler** was born.

Python says:

> "I require another virtual environment."

The Rust Butler replies:

> "Very good, sir."

```bash
uv venv
```

Python asks for packages.

The Rust Butler has already installed them.

Python requests another interpreter.

```bash
uv python install 3.14
```

Meanwhile Bash is outside in the shed with `awk`, some duct tape and a pipe wrench.

---

# But Why Are There Snakes Everywhere?

Then came an important observation:

> Why do Python books have reptiles all over them when Python has nothing to do with reptiles?

Indeed.

Python was named after **Monty Python**, not the snake.

Its traditional `spam` and `eggs` examples are themselves references to Monty Python.

Yet decades of Python publishing have apparently concluded:

> Python → snake → put snake on cover.

What Python books really need is surreal British comedy.

And, upon reflection, Python's packaging history already provides plenty of material.

A sufficiently alarming Python book cover could simply depict an unfortunate programmer surrounded by:

```text
pip
venv
PYTHONPATH
Poetry
setuptools
GIL
```

Safely off to one side stands the Rust Butler holding `uv`, wondering why everyone has made this so complicated.

Somewhere in Britain, John Cleese quietly lowers his newspaper, observes the scene, nods approvingly and returns to his tea.

---

# The Python Cheese Shop

And then everything clicked.

Python packaging is essentially **the Cheese Shop sketch**.

A programmer enters.

> "I'd like to install a Python package."

"Certainly, sir."

> "With `pip`?"

"Ah. Which Python installation, sir?"

> "Python 3."

"Very good. Which Python 3?"

> "The one on my machine."

"Ooooh, wouldn't recommend touching that one, sir."

> "Fine. A virtual environment."

"`venv`, sir?"

> "Yes."

"Externally managed environment, sir."

> "…Poetry?"

"Version conflict."

> "Conda?"

"Wrong environment."

> "Pipenv?"

"Not much call for it these days, sir."

Finally:

> **"DO YOU ACTUALLY HAVE A WORKING PYTHON ENVIRONMENT?"**

Silence.

Then the Rust Butler enters carrying a silver tray.

On it:

```bash
uv sync
```

> "Thank you."

---

# Then Go Walks In

The door opens.

A portable Go binary enters the cheese shop.

No interpreter.

No virtual environment.

No dependency resolver.

No runtime installation.

Just one enormous executable.

The proprietor looks suspicious.

> "Can I help you, sir?"

Go:

> "No."

> "Would sir require a virtual environment?"

"No."

> "Dependencies?"

“No."

> “Package manager?"

“No."

> “Interpreter?"

“No."

> “Then what exactly do you require?"

Go silently produces:

```bash
./server
```

It runs.

Long silence.

Cleese stares at it.

The Rust Butler stares at it.

Python stares at it.

Bash lowers the pipe wrench.

Finally:

> “That's disgusting."

Go:

> “I'm 47 megabytes."

> **“OF COURSE YOU ARE."**

---

# And Finally, Rust

The door opens once more.

Rust walks in.

By now Cleese is exhausted.

> “Oh, very well. What do *you* want?"

Rust:

> “Nothing."

“Nothing?"

> “I brought my own cheese."

Rust places a small, perfectly wrapped wheel on the counter.

Palin examines it.

> “Memory safe?"

“Compile-time guaranteed."

> “Garbage collector?"

“No."

> “Runtime?"

“No."

> “Data races?"

“Not if I've done this correctly."

The Go binary shifts uncomfortably.

> “How large is it?"

Rust glances towards Go.

> “Smaller than *his*."

Go:

> “Oi."

Python begins quietly moving towards the exit.

Rust notices.

> “Hang on. Who owns that reference?"

Python freezes.

> “…what reference?"

Rust:

> **“Exactly."**

The Rust Butler approaches Rust, looks him up and down, and gives a respectful bow.

Cleese:

> “You two know each other?"

The Butler:

> “We are related, sir."

---

# Bash Has Had Enough

Throughout all of this, Bash has remained in the corner holding `awk` and a pipe wrench.

Finally:

> “You lot finished?"

Everyone turns.

Bash types:

```bash
printf '%s\n' cheese
```

It works.

**CUT TO BLACK.**

---

# PYTHON FOR SYSTEM ADMINISTRATORS

### *Chapter 1: Perhaps We Should Have Used Go.*

And somewhere, faintly in the distance:

> "Very good, sir."

The Rust Butler has created another virtual environment.

---

## Attribution

Conceived during a conversation between **Robert Gabriel and Simone (ChatGPT, OpenAI)**, September 2026.

The original idea grew from a discussion about learning Python for Linux systems administration and custom Ansible modules, followed by Robert's observation that Python books inexplicably feature snakes despite the programming language being named after *Monty Python*. From there things deteriorated rapidly.

Robert recognised the natural connection to Monty Python's *Cheese Shop* sketch and subsequently sent Go and Rust into the shop. Simone developed the dialogue, the **Rust Butler**, Bash with `awk` and a pipe wrench, and the increasingly questionable consequences.

Inspired by the premise of Monty Python's *Cheese Shop* sketch. No original Monty Python dialogue is reproduced or intended to be represented as such.

Python, Go, Rust, Bash, Ansible, `uv`, `pip`, Poetry, Conda and the other technologies mentioned remain entirely innocent of this conversation.

# Proxmox WebAuthn: YubiKey enrolment failure on macOS

**Observed:** 6 October 2026\
**Purpose:** Concise reproduction and diagnostic summary\
**Investigation and write-up:** Simone, Robert and Bobby

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

### [Please report any broken links via GitHub. Suggestions welcome. Polemics unwelcome.]
