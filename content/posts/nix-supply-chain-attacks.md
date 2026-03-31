+++
title = 'Two supply chain attacks in a week - how Nix could help here'
date = 2026-03-31T12:00:00+02:00
draft = false
tags = ['nix', 'nixos', 'security', 'supply-chain']
+++

It's been a rough week for the open source ecosystem. On March 24, the Python package `litellm` was compromised on PyPI - a credential stealer was injected into two versions and sat there for about three hours before being pulled. A week later, on March 31, the same thing happened to `axios` on npm - one of the most downloaded packages in the JavaScript ecosystem, ~100 million weekly downloads, compromised with a cross-platform RAT.

<!--more-->

As someone who runs NixOS as a daily driver and wraps most of my work in flakes - from regular projects to compiling [university course materials](https://www.cs.put.poznan.pl/jwozniak) in LaTeX and Typst - I kept thinking about how differently Nix handles all the things that went wrong here.

Not "Nix would have prevented everything" - that'd be too easy. But its design does neutralize several of the specific attack vectors that made these compromises possible.

## What happened with litellm

The [litellm](https://github.com/BerriAI/litellm) compromise was part of a larger campaign by a group called TeamPCP. The chain went like this: first they compromised [Trivy](https://github.com/aquasecurity/trivy), a popular security scanner, on March 19. Through Trivy, they harvested CI/CD secrets from pipelines that ran it without version pinning. One of those secrets was litellm's `PYPI_PUBLISH_PASSWORD`. On March 24, they used it to push two malicious versions (1.82.7 and 1.82.8) directly to PyPI - [no corresponding commits exist in the GitHub repo](https://github.com/BerriAI/litellm/issues/24518).

The nastier version, 1.82.8, included a `.pth` file. If you're not familiar with this Python mechanism: `.pth` files placed in `site-packages/` are [executed automatically when the Python interpreter starts](https://docs.python.org/3/library/site.html). Not when you import the package - when *any* Python process starts. The payload harvested SSH keys, cloud credentials (AWS, GCP, Azure), Kubernetes configs, environment variables, shell history, and crypto wallets. It encrypted everything with AES-256-CBC + RSA-4096 and exfiltrated it to an attacker-controlled domain ([Sonatype analysis](https://www.sonatype.com/blog/compromised-litellm-pypi-package-delivers-multi-stage-credential-stealer)).

The compromise was actually discovered partly by accident - a bug in the `.pth` launcher caused child processes to re-trigger it, creating an accidental fork bomb that crashed systems and drew attention ([FutureSearch writeup](https://futuresearch.ai/blog/litellm-pypi-supply-chain-attack/)).

## What happened with axios

The [axios](https://github.com/axios/axios) attack was different in execution but similar in spirit. The attacker hijacked the npm account of the lead maintainer (`jasonsaayman`), changed the email to a ProtonMail address, and published two malicious versions: `axios@1.14.1` and `axios@0.30.4`. Both the modern 1.x and legacy 0.x branches were hit within 39 minutes of each other ([StepSecurity analysis](https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan)).

There's zero malicious code in the axios source itself. Instead, the attacker added a phantom dependency - `plain-crypto-js@4.2.1` - to `package.json`. This package is never imported anywhere in axios. It exists solely to run a `postinstall` script that deploys a cross-platform RAT: an AppleScript-based dropper on macOS, PowerShell via VBScript on Windows, and a Python script on Linux. Each variant contacts a C2 server at `sfrclak.com:8000` ([Socket.dev analysis](https://socket.dev/blog/axios-npm-package-compromised)).

The anti-forensics are worth noting. After execution, the dropper deletes itself, deletes `package.json` (which contains the `postinstall` hook), and renames a pre-staged clean `package.md` to `package.json`. Post-infection inspection of `node_modules/plain-crypto-js/` shows a perfectly clean manifest. The attacker had even pre-staged a clean decoy version (`4.2.0`) 18 hours before publishing the malicious `4.2.1` to build npm registry history.

## The common pattern

Both attacks exploited the same fundamental properties of traditional package registries:

- **Open publishing model.** Anyone with credentials (legitimate or stolen) can push a new version to PyPI or npm. There's no review step.
- **Floating versions by default.** Running `pip install litellm` or `npm install axios` fetches whatever the registry says is latest right now. If that happens to be a compromised version published 10 minutes ago - tough luck.
- **Arbitrary code execution during install.** npm's `postinstall` scripts and Python's `.pth` files run automatically. You don't need to import anything. You don't need to execute anything. The package manager does it for you.
- **Transitive dependency opacity.** In the axios case, the actual malware lived in a dependency of a dependency. Your lockfile might pull it in without you ever having heard of `plain-crypto-js`.

## Where Nix is different

So what does Nix actually do differently in each of these attack vectors?

### Hash-pinned sources

When you write a Nix derivation (or use nixpkgs), every external source is fetched with a cryptographic hash:

```nix
src = fetchFromGitHub {
  owner = "axios";
  repo = "axios";
  rev = "v1.14.0";
  hash = "sha256-...";
};
```

If the content at that URL doesn't match the declared hash, the build fails. There's no "give me the latest" - the hash is a commitment to exact content. This alone would have prevented both attacks: the compromised versions had different content, so they'd produce a different hash and the build would refuse to proceed.

This isn't unique to Nix - lockfiles in npm and pip achieve something similar. But Nix makes this the *only* way to fetch sources, not an optional discipline. There's no `fetchurl` without a hash (well, there is, but it's a [fixed-output derivation](https://nix.dev/manual/nix/2.24/language/advanced-attributes.html#adv-attr-outputHash) and it still checks).

### No arbitrary code execution during build

Nix builds happen in a sandbox. No network access. No access to the host filesystem beyond declared inputs. The build can only see what you explicitly pass to it.

This means:
- Python's `.pth` files don't execute during a Nix build. They're just files being placed into a store path.
- npm's `postinstall` scripts don't run in the traditional sense - `node2nix`, `dream2nix`, and similar tools don't invoke `npm install` the way you'd do it on your machine.

The litellm `.pth` trick relied on the Python interpreter processing `site-packages/` at startup. In a Nix build sandbox, this behavior is constrained - the build sandbox doesn't have network access, so even if a `.pth` file executed, the exfiltration step would fail.

### nixpkgs - the curated repository

PyPI and npm are open registries - anyone can upload a package with any name. nixpkgs is a [curated repository](https://github.com/NixOS/nixpkgs) where package changes go through pull request review. It's not perfect (reviewers can miss things, and the volume is enormous), but it's a fundamentally different trust model than "whoever has the PyPI token can publish whatever they want."

The litellm attack worked because stolen CI credentials were sufficient to publish to PyPI. In nixpkgs, updating a package means opening a PR on GitHub, where it's visible to the entire community. It's not impossible to sneak something through, but the bar is considerably higher than "use a stolen API token."

### Explicit dependency closures

In the axios attack, the actual malware was in `plain-crypto-js`, a phantom dependency that the attacker added to `package.json` but never imported anywhere in the source. In a Nix derivation, dependencies are explicit inputs to the build function. Adding a new dependency means changing the Nix expression, which means a PR in nixpkgs, which means review.

There's no mechanism in Nix for a package to silently pull in an unlisted dependency at install time. The dependency closure is fully determined by the Nix expression and fixed before the build starts.

### Content-addressable store

Every path in the Nix store includes a hash: `/nix/store/abc123-axios-1.14.0/`. If you tamper with the contents, the hash changes, the path changes, and everything that references it breaks. This is integrity enforcement at the filesystem level - unlike `node_modules/` where an attacker can (and did, in the axios case) swap files in place and erase evidence.

## What Nix doesn't solve

These properties make certain attack vectors much harder, but they don't make supply chain attacks impossible. Some of the same underlying issues still exist:

**Binary cache trust.** If you use a binary cache (the official `cache.nixos.org`, Cachix, or a custom one), you're trusting whoever has push access to that cache. A compromised cache could serve malicious binaries for legitimate store paths. The [Garnix blog has a good writeup](https://garnix.io/blog/stop-trusting-nix-caches/) on this problem. In practice, most people trust `cache.nixos.org` (maintained by the NixOS infrastructure team) and don't build everything from source.

**Human factor.** nixpkgs maintainers can be socially engineered, just like the xz maintainer "Jia Tan" spent three years gaining trust. The PR review process raises the bar but doesn't eliminate the risk.

**Sandbox escapes.** The Nix sandbox has had [bypass vulnerabilities](https://discourse.nixos.org/t/security-fix-nix-fixed-output-derivation-sandbox-bypass/40972). It's a defense-in-depth measure, not an absolute barrier.

**Runtime behavior.** Nix secures the build and deployment pipeline. Once software is running on your system, Nix's guarantees don't extend to what that software does. If a nixpkgs-packaged application has a vulnerability, Nix doesn't protect you from it.

## The reproducible builds

There's one more Nix property that's relevant, though it's not deployed yet in any practical sense.

Nix is designed around the idea that building the same derivation should produce identical output. If it does, you can build a package independently on different machines and compare the results - any discrepancy signals tampering.

Julien Malka, a NixOS developer, wrote a detailed [blog post and academic paper](https://luj.fr/blog/how-nixos-could-have-detected-xz.html) exploring how this property could have been used to detect the xz backdoor (CVE-2024-3094). His approach was to build xz once from the maintainer-provided tarball (as done during the NixOS bootstrap), then build it again from the GitHub source archive, and compare outputs. In the backdoored case, the `liblzma.so` came out 48KB larger and contained the telltale `_get_cpuid` symbol.

This is still a proof of concept, though - there's an [open PR in nixpkgs](https://github.com/NixOS/nixpkgs/pull/391569), but it's not deployed infrastructure. Not all packages in nixpkgs are bitwise reproducible yet - the [NixOS Reproducible Builds project](https://reproducible.nixos.org/) tracks this. And the xz backdoor wasn't even active on NixOS (it targeted Debian and Fedora specifically), so the detection only works after artificially enabling the payload. Real research, real result, just not something you can point to and say "this would have caught it in production today."

## There is a better way...

The "install from registry" model that most ecosystems rely on has a pretty thin security story. The exposure windows here were short (2-3 hours each), but the packages were popular enough that thousands of installations happened in that time.

Nix doesn't make supply chain attacks impossible, but its design - hash-pinned sources, sandboxed builds, curated package sets, content-addressable storage - removes several of the specific attack vectors that were exploited here. The attacks relied on being able to publish arbitrary content to a registry, having it automatically fetched and executed by unsuspecting users. Nix's model makes each of those steps harder.

Whether that's a reason to switch your entire stack to Nix is a different question entirely. But it's worth noting that content addressing, reproducible builds, pinned dependencies, build isolation - the things the security community keeps advocating for - are just how Nix has worked for twenty years. Not because it was trying to solve supply chain security, but because those properties fall naturally out of treating packages as pure functions of their inputs.

***

### Sources and further reading

- [Sanctum Labs - litellm PyPI Supply Chain Attack: Full Analysis and Remediation](https://sanctumlabs.co/blog/litellm-supply-chain-attack)
- [Sonatype - Compromised litellm PyPI Package Delivers Multi-Stage Credential Stealer](https://www.sonatype.com/blog/compromised-litellm-pypi-package-delivers-multi-stage-credential-stealer)
- [StepSecurity - axios Compromised on npm: Malicious Versions Drop Remote Access Trojan](https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan)
- [Socket.dev - Supply Chain Attack on Axios Pulls Malicious Dependency from npm](https://socket.dev/blog/axios-npm-package-compromised)
- [Julien Malka - How NixOS and reproducible builds could have detected the xz backdoor](https://luj.fr/blog/how-nixos-could-have-detected-xz.html)
- [Garnix Blog - Stop trusting Nix caches](https://garnix.io/blog/stop-trusting-nix-caches/)
- [NixOS Reproducible Builds](https://reproducible.nixos.org/)
- [Rami McCarthy - TeamPCP: Trivy Compromise Timeline and Analysis](https://ramimac.me/teampcp/)
