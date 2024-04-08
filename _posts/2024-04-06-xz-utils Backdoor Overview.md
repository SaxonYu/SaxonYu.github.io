---
layout: post
title: "From Compression to Compromise: xz-utils Backdoor Overview"
---
On March 29, 2024, an upstream supply-chain poisoning targeting a popular file compression tool was uncovered by Andres Freund. Versions 5.6.0 and 5.6.1 of xz-utils, after installation, contain a backdoor ([CVE-2024-3094](https://nvd.nist.gov/vuln/detail/CVE-2024-3094)). The actual attack path is extremely complicated, involving sophisticated social engineering, code obfuscation, and advanced evasion techniques such as function hooking, sandbox escaping, etc. In the rest of the article,  I'll delve into the sophistication of the social engineering tactics employed and provide a high-level summary of the technical details. There's a lot to uncover, so I plan to circle back with a deeper technical exploration in a future post. There’s plenty to learn and share!

# Background: 
The mastermind responsible for planting the backdoor uses the pseudonyms Jia Tan and JiaT75 on GitHub, dedicated over two years to a stealthy operation. By submitting legitimate bug fixes and employing social engineering, Jia Tan gradually took over and started embedding malicious code snippets into the xz-utils project. The infiltration remained undetected until March 29, 2024, when Andres Freund, a vigilant Microsoft employee, uncovered the backdoor.

## Infiltration Operation
The GitHub account JiaT75 was first created in 2021. However, it was not until March 2022, Jia Tan started to [show interest](https://www.mail-archive.com/xz-devel@tukaani.org/msg00540.html) in the xz-utils project. The next turning point is around April 2022. After Jia Tan submitted another patch via the mailing list. It was then a user named [Jigar Kumar](https://www.mail-archive.com/search?l=xz-devel@tukaani.org&amp;q=Kumar&amp;x=0&amp;y=0) joined the conversation implying that Jia Tan is doing a great job, but unfortunately, the whole project seemed to be in a state of stagnation. These messages are yet still complaints but will quickly become more confrontational, particularly towards the project maintainer, Lasse Collin.
![Kumar1]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/Kumar1.png){:style="display:block; margin-left:auto; margin-right:auto"}

On May 19, 2022, A query from [Dennis Ens](https://www.mail-archive.com/xz-devel@tukaani.org/msg00562.html) regarding whether the xz for Java was still maintained got a prompt affirmation from Lasse Collin saying Jia Tan might have a bigger role in the future.
![Ens1]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/Ens1.png){:style="display:block; margin-left:auto; margin-right:auto"}

After a couple weeks, tension escalated when Jigar Kumar began to assert pressure on Lasse Collin, suggesting it was time to find a new project maintainer. Even though Lasse excused himself because of mental health issues and restated the presence of Jia Tan, Jigar Kumar persisted, advocating for a change of maintainer.![Ens2]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/Ens2.png){:style="display:block; margin-left:auto; margin-right:auto"}

Although heated discussion is not unusual in the realm of open-source projects, everything is coincidental that both Jigar Kumar and Dennis Ens only existed within the discussion of XZ, and no other related accounts were found. Furthermore, Dennis Ens’s reply to Lasse Collin, expressing deep apologies about the mental health issues while suggesting a change of leadership, added another layer to the dynamic. It made me wonder if a "good cop, bad cop" strategy was at play.![Ens3]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/Ens3.png){:style="display:block; margin-left:auto; margin-right:auto"}

## Trust to Trojan
The involvement of Jigar Kumar and Dennis Ens, whether coincidental or not, subtly paved the way for Jia Tan's gradual control over the project. With more and more control over the project, Jia Tan did not upload the whole trojan all at once but strategically staged them among other distractions to avoid suspicion. Despite these smoke bombs, several pivotal actions stand out:
1. On Mar 20, 2023, Jia Tan [changed xz project's primary contact](https://github.com/JiaT75/oss-fuzz/commit/6403e93344476972e908ce17e8244f5c2b957dfd) within Google's oss-fuzz service to his own. To whoever not familiar with Google's oss-fuzz, it's a free service that runs fuzzers for open source projects and privately alerts developers to the bugs detected. Changing the main contact means that any potential issues related to xz project will be directed to Jia Tan himself obfuscating the detection of any backdoor he intended to implement.![Trust1]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/Trust1.png){:style="display:block; margin-left:auto; margin-right:auto"}
2. According to security researchers all over the world, part of the exploit code was heavily obfuscated and included within the test files in the Git Repository, uploaded through two commits on [February 23, 2024](https://git.tukaani.org/?p=xz.git;a=commitdiff;h=cf44e4b7f5dfdbf8c78aef377c10f71e274f63c0), and [March 8, 2024](https://git.tukaani.org/?p=xz.git;a=commitdiff;h=6e636819e8f070330d835fce46289a3ff72a7b89).![Trust2]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/Trust2.png){:style="display:block; margin-left:auto; margin-right:auto"}
3. Towards the end of March 2024, there was a sudden push for the newer version of xz-utils, signaling that whoever is behind was preparing for a final, large-scale attack.![Trust3]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/Trust3.png){:style="display:block; margin-left:auto; margin-right:auto"}

## Here Comes the Sherlock 
Andres Freund, who was conducting micro-benchmarking at the time, noticed an unusual use of CPU time in the sshd process, particularly with liblzma, a component of the xz-utils package. After a careful analysis, Andres Freund filed a report to oss-security revealing the backdoor. It's surprising that what seemed to be minor Valgrind errors and a sudden increase in CPU usage were actually indicators of a malicious backdoor. Freund, not being a security researcher or malware analyst, made an incredible discovery from details that many would have easily overlooked.![Sherlock1]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/Sherlock1.png){:style="display:block; margin-left:auto; margin-right:auto"}
# Technical Overview:
## Basic Information
- The actual exploit code is heavily obfuscated and resides in the test files in the `tests/` folder within the Git Repository. Meanwhile, the script `build-to-host.m4` deobfuscate and unpacks the exploit code is included within the released tarball.

- The exploit code eventually produces a compromised `liblzma` which is used by `libsystemd`. Consequently, whenever the Secure Shell Daemon (`sshd`, commonly implemented with OpenSSH on Unix-like systems) is configured to use `systemd` for service notifications, it inadvertently triggers the loading of the tampered `liblzma`. This, in turn, interferes with and modifies the SSH authentication process.![backdoor1]({{ '/' | relative_url }}public/screenshots/xz-utils_Backdoor_Overview/backdoor1.png){:style="display:block; margin-left:auto; margin-right:auto"}

- It was originally considered as a SSH authentication bypass but with [further analysis](https://bsky.app/profile/filippo.abyssdomain.expert/post/3kowjkx2njy2b), the backdoor actually enables remote code execution(RCE), but only for the attacker who installed the backdoor.

- The exploit leverages `IFUNC`, a GNU C Library (`glibc`) feature designed for resolving function calls to the most appropriate implementation at runtime. However, within the exploit code, it is repurposed for function hooking/redirection to hijack OpenSSH's authentication routine.

## High-Level Execution Flow
1. The `build-to-host.m4` script initiates the recovery of the corrupted archive `bad-3-corrupt_lzma2.xz`. The successful unarchiving process yields a bash script, effectively serving as the first payload delivery mechanism in this sophisticated exploit chain.

2. Running this first-stage bash script triggers the extraction of a second bash script from the file `good-large_compressed.lzma`.

3. The final stage in this intricate attack sequence involves running the second bash script to extract the shared object file `liblzma_la-crc64-fast.o`. This object is then compiled into the compromised `liblzma` library. 

# Conclusion: 
The xz-utils backdoor is the most sophisticated supply-chain attack known to the date. If it wasn't for Andres Freund, the potential ramifications for the open-source ecosystem could have been catastrophic. 

It is also heartening to witness the global security research community rally to the cause. Together, they are dissecting the attack, sharing insights, and fortifying defenses, ensuring that knowledge gleaned from this breach will serve as a reinforcement against future threats.

# Reference
- [https://boehs.org/node/everything-i-know-about-the-xz-backdoor](https://boehs.org/node/everything-i-know-about-the-xz-backdoor)
- [https://pentest-tools.com/blog/xz-utils-backdoor-cve-2024-3094](https://pentest-tools.com/blog/xz-utils-backdoor-cve-2024-3094)
- [https://www.wiz.io/blog/cve-2024-3094-critical-rce-vulnerability-found-in-xz-utils](https://www.wiz.io/blog/cve-2024-3094-critical-rce-vulnerability-found-in-xz-utils)
- [https://gist.github.com/thesamesam/223949d5a074ebc3dce9ee78baad9e27?permalink_comment_id=5005888](https://gist.github.com/thesamesam/223949d5a074ebc3dce9ee78baad9e27?permalink_comment_id=5005888)
