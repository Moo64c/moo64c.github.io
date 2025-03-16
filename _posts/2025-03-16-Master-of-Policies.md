---
title: "Manager of Policies"
author: "Amir Arbel"
categories: ["Devlog"]
date: "2025-03-16"
tags: ["Trusteer", "Lua", "Fraud-detection", "low-code"]
excerpt: "Bridging the gap between known and new fraud threats. Empowering client to rapidly respond to threats - without writing a single line of code."
---
Sophisticated, new emerging fraud attacks spread every few minutes, leaving customers vulnerable. Trusteer Pinpoint defends against known frauds, but responding to new threats can take days, even weeks. What if we could empower customers to block those threats in mere hours? That's the vision behind *Policy Manager*, a project I was the lead developer on the Pinpoint team. We've revolutionized fraud response, empowering customers with rapid, direct control.


In early November, a select group of our customers were introduced to the new *Policy Manager*. With just a few clicks in Trustboard (Trusteer's customer-facing toolkit), they could now create and instantly publish custom rules, directly overriding Pinpoint's risk assessments.

![Respond quickly to wide fraud attacks using a simple UI.](https://moo64c.github.io/assets/images/Master-of-Policies/visualization.jpeg)

This new capability enabled instant deployment of alerts and automated actions on emerging threats, as soon as they were detected. Previously, a customer-identified threat would trigger a lengthy, multi-team process for analysis, rule creation and deployment. During this delay, customers remained vulnerable to known threats. *Policy Manager* eliminates this friction, enabling Pinpoint to react swiftly and empower customers to protect their own users.

*Policy Manager*'s impact was immediate. Within days, it began alerting on suspected fraud. A month later, a customer experienced a critical scenario: a previously blocked attack bypassed detection due to a configuration issue. Using Policy Manager, they swiftly identified and blocked the attack in **just three hours**, showcasing the tool's power in real-world situations.

![Stopping a real fraud in the wild, a month from release.](https://moo64c.github.io/assets/images/Master-of-Policies/real-world-use.png)

Although Trusteer's Support team provided guidance for this specific rule, the UI's self-service functionality enables customers to create and deploy rules autonomously, unlocking the potential for even faster response times.

### Fast, Friction-Free, Fraud Fighting
Policy Manager delivers on its promise of speed and simplicity. Through a new, user-friendly interface in Trustboard, customers create rules that are translated into efficient Lua code. This code is then rapidly deployed across Trusteer's systems using S3. When Pinpoint is called to assess risk, *Policy Manager*'s rules are applied, ensuring the most up-to-date protection. This seamless integration ensures that customers get the most accurate risk assessment possible.

Simplicity drove the design of Trustboard's *Policy Manager* UI. The no-code interface enables the customer's fraud analysts to create custom rules without programming, focusing on rapid threat identification and response. Crucially, *Policy Manager* leverages the existing communication channel with Pinpoint. This removes additional development on the customer's side - preserving their existing alert and response workflows.

Within _Rules Engine_, the instructions received are first validated as JSON to ensure they adhere to a specific structure. Then, they're transformed into Lua code, undergoing rigorous checks for correct syntax and type safety. Performance optimizations are applied during this code generation. This automated process allows us to understand the precise requirements of each rule, enabling us to design efficient and reliable code.

![A bit of the templating engine. Uses a formal language definition to create flexible code blocks.](https://moo64c.github.io/assets/images/Master-of-Policies/formal-language.png)

_Policy Manager_ leverages the same dataset as _Policy_, which runs before itm eliminating the need for extra database calls and ensuring optimal performance. It executes rules within a secure sandbox environment, with strictly defined inputs and outputs. When a rule is triggered in Policy Manager, the resulting risk assessment is updated accordingly. This code generation approach enables us, as developers, to continuously optimize performance and add new capabilities without disrupting the production system.

![Rules Engine in Pinpoint and *Policy Manager* in Trustboard work together to catch fraud, fast.](https://moo64c.github.io/assets/images/Master-of-Policies/two-sides.jpeg)

### One Engine To Rule Them

From a product perspective, *Policy Manager* is the user-facing innovation, while the *Rules Engine* is its powerful backend. This engine, initially conceived years ago within Pinpoint's Hub Service, remained largely unused. However, it found new life in Q3 2024, undergoing significant adaptation for integration with Pinpoint, alongside the development of Policy Manager. Both were launched in Q4 for a select group of early adopters.

Originally, the *Rules Engine* boasted a wider range of features, including tiered rulesets, test modes, and granular control. While some features were streamlined for the initial Pinpoint integration, they remain a part of its future potential, awaiting the right use case.

#### Rule Wrangling
The Trustboard team developed an intuitive and streamlined _No-code_ UI, directly accessible to customers within Trustboard. This interface allows users to create rules within a *ruleset*, where they can define conditions, set recommendations and scores, and prioritize rules with ease.

![Creating a rule in Policy Manager's UI.](https://moo64c.github.io/assets/images/Master-of-Policies/rule.png)

Users interact with a simplified interface, completely removed from the complexities of the underlying code. Details like variable types, parameter orders, and error handling are handled behind the scenes, within the templating engine and secure sandbox.

![Setting order, enabling and disabling rules in Policy Manager's UI.](https://moo64c.github.io/assets/images/Master-of-Policies/ruleset.png)

At present, customers can utilize a single ruleset affecting all applications. Upon a 'publish' action, the UI assembles the instructions, and the _Rules Engine_ compiles and distributes the code to the appropriate Pinpoint clusters in minutes.

### What's Next?

Policy Manager is a powerful tool today, with optimized code generation, rapid global deployment, and ease of use for non-programmers. However, we're committed to continuous improvement and expansion.

Our vision includes even faster deployment, cost reduction by eliminating S3, expanded capabilities for checking session attributes, dry/wet and tiered rulesets, list comparisons, and much more.

### Thanks
Thank you for taking the time to learn about Policy Manager. This project was a true team effort, made possible by the dedication and expertise of many individuals, including Liran Tiebloom (TPM), Vikas Shankar Poojary and Anat Ilani (QA), Nati Braha, Gal Amuyal, Or Krichli, and Yehuda Fireberger (Trustboard), and my colleagues Adi Meiman and Nir Nahum from the Pinpoint team, alongside myself, Amir Arbel.
