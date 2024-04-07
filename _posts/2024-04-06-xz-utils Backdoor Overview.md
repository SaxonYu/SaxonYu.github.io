---
layout: post
title: "From Compression to Compromise: xz-utils Backdoor Overview"
---
On March 29, 2024, an upstream supply-chain poisoning targeting xz-utils, a popular file compression tool, was uncovered by Andres Freund. Versions 5.6.0 and 5.6.1 of xz-utils, after installation, contain a backdoor ([CVE-2024-3094](https://nvd.nist.gov/vuln/detail/CVE-2024-3094)). The actual attack path is extremely complicated, involving sophisticated social engineering, code obfuscation, and advanced evasion techniques such as function hooking, sandbox escaping, etc. In the rest of the article,  I'll delve into the sophistication of the social engineering tactics employed and provide a high-level summary of the technical intricacies. There's a lot to uncover, so I plan to circle back with a deeper technical exploration in a future post. There’s plenty to learn and share!

# Background: 
The mastermind responsible for planting the backdoor uses the pseudonyms Jia Tan and JiaT75 on GitHub, dedicated over two years to a stealthy operation. By submitting legitimate bug fixes and employing social engineering, Jia Tan gradually took over and started embedding malicious code snippets into the xz-utils project. The infiltration remained undetected until March 29, 2024, when Andres Freund, a vigilant Microsoft employee, uncovered the backdoor.
## Infiltration Operation
The GitHub account JiaT75 was first created in 2021.
## Trust to Trojan
## Here Comes the Sherlock 

# Technical Details: 

# Conclusion: 

