# Gibbon SIS Data API

API integration hub for [Gibbon SIS](https://gibbonedu.org/): a read-only [1EdTech OneRoster
1.1](https://www.1edtech.org/standards/oneroster) REST API plus write-capable User Provisioning
(OneSync-compatible) for syncing rosters and user accounts with OneRoster-compatible consumers.

## Disclaimer

This project uses the [1EdTech OneRoster](https://www.1edtech.org/standards/oneroster) standard.
Neither this extension nor its author is affiliated with, endorsed by, or certified by 1EdTech
Consortium, Inc.

This extension is licensed under Apache License 2.0 (see [LICENSE](./LICENSE)), independently of
Gibbon core, which is licensed under GNU GPLv3. If you have your own concerns about combining
software under these two licenses, please make your own determination — this is not legal advice.

**This module reads and can write real student and staff data.** Review the source code and
understand what it does before deploying it against a live Gibbon instance. Test against a
non-production copy of your data first, especially before enabling User Provisioning (OneSync)
write access, which can create, modify, and deactivate real user accounts.

**You are responsible for your own compliance obligations** — FERPA, GDPR, or whatever data
protection law applies to your school or district — when using this module to expose or modify
student and staff data. This is a technical integration tool, not a compliance product; it does
not make any determination about whether a given use is compliant with your own legal obligations.

**No warranty, no liability.** This software is provided "AS IS," without warranty of any kind,
express or implied — see [LICENSE](./LICENSE) (Apache License 2.0) for the full legal terms,
particularly Sections 7 and 8. In short: use this at your own risk, and the author is not liable
for any damages, data loss, or other consequences arising from its use or misuse.

## Installation

See [INSTALL.md](./INSTALL.md) for step-by-step setup instructions, including hosting
requirements, credential generation, and troubleshooting.
