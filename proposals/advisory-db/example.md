```toml

[advisory]
# Identifier for the advisory (mandatory). Will be assigned a "HSEC-YYYY-NNNN"
# identifier e.g. HSEC-2022-0001. Please use "HSEC-0000-0000" in PRs.
id = "HSEC-0000-0000"

# Name of the affected package on Hackage (mandatory)
package = "mycrate"

# Disclosure date of the advisory as an RFC 3339 date (mandatory)
date = 2021-01-31

# URL to a long-form description of this issue, e.g. a GitHub issue/PR,
# a change log entry, or a blogpost announcing the release (optional)
url = "https://github.com/mystuff/package/issues/123"

# Optional: Categories this advisory falls under. Valid categories are:
# "code-execution", "crypto-failure", "denial-of-service", "file-disclosure"
# "format-injection", "memory-corruption", "memory-exposure", "privilege-escalation"
categories = ["crypto-failure"]

# Optional: a Common Vulnerability Scoring System score. More information
# can be found on the CVSS website, https://www.first.org/cvss/.
#cvss = "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H"

# Freeform keywords which describe this vulnerability (optional)
keywords = ["ssl", "mitm"]

# Vulnerability aliases, e.g. CVE IDs (optional but recommended)
# Request a CVE for your HSec vulns: https://iwantacve.org/
#aliases = ["CVE-2018-XXXX"]

# Related vulnerabilities (optional)
# e.g. CVE for a C library wrapped by a Haskell library
#related = ["CVE-2018-YYYY", "CVE-2018-ZZZZ"]

# Optional: metadata which narrows the scope of what this advisory affects
[affected]
# CPU architectures impacted by this vulnerability (optional).
# Only use this if the vulnerability is specific to a particular CPU architecture,
# e.g. the vulnerability is in x86 assembly.
# For a list of CPU architecture strings, see the documentation for System.Info.arch:
# <https://hackage.haskell.org/package/base-4.16.1.0/docs/System-Info.html>
#arch = ["x86", "x86_64"]

# Operating systems impacted by this vulnerability (optional)
# Only use this if the vulnerable is specific to a particular OS, e.g. it was
# located in a binding to a Windows-specific API.
# For a list of OS strings, see the documentation for System.Info.os:
# <https://hackage.haskell.org/package/base-4.16.1.0/docs/System-Info.html>
#os = ["mingw32"]

# Table of canonical paths to vulnerable declarations in the package (optional)
# that describes which versions impacted by this advisory used that particular
# name (e.g. if an affected function or datatype was renamed between versions). 
# The path syntax is the module import path, without any type signatures or
# additional information, followed by the affected versions.
#declarations = { "Acme.Broken.function" = ">= 1.1.0 && < 1.2.0", "Acme.Broken.renamedFunction = ">= 1.2.0 && < 1.2.0.5"}

# Versions affected by the vulnerability
[versions]
affected = ">= 1.1.0 && < 1.2.0.5"
```

# Acme Broken's Use of `unsafePerformIO` is Unsound

Acme Broken implements safe internal mutation using `unsafePerformIO`. However, in a multithreaded context, an attacker can cause a service to return the wrong answer.


## Mitigations

 * Don't use concurrency in applications that use Acme Broken
