# WSL2 Wireless Kernel

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](LICENSE)

Guides to building a custom WSL2 kernel with `mac80211_hwsim` and wireless support, so that wireless
emulators such as [Mininet-WiFi](https://github.com/intrig-unicamp/mininet-wifi) and MiniNDN-WiFi run
directly on Windows 10/11, without dual boot or a virtual machine.

## Guides

| Mode | Status | Beginner | Advanced |
|---|---|---|---|
| IBSS (ad hoc) | Ready | [ibss/beginner.md](ibss/beginner.md) | [ibss/advanced.md](ibss/advanced.md) |
| Mesh (802.11s, HWMP) | Coming soon | | |
| AP/STA with encryption | Out of scope, by design | | |

The **beginner** guide explains the reason behind every kernel option. The **advanced** guide lists
the steps and options only.

The default WSL2 kernel is compiled with the bare minimum, and these guides follow the same lean
principle: they enable only what the covered mode needs. Examples that use other modes may not run.

## How to cite

If you use these guides in your work, please cite them as described in [`CITATION.cff`](CITATION.cff)
(GitHub shows it as "Cite this repository").

## License

The guides are licensed under the
[Creative Commons Attribution 4.0 International License (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
See [`LICENSE`](LICENSE). Scripts added to this repository are licensed under the MIT license, stated
in each script.

These guides were first published as a [GitHub gist](https://gist.github.com/brunosfer/deec7b4d941b1afad83b597de9a0f3a3), which now points here.

Questions and bug reports are welcome as [GitHub issues](https://github.com/brunosfer/wsl2-wireless-kernel/issues).
