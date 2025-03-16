---
title:  "Manager of Policies"
author: "Amir Arbel"
categories: ["Devlog", "Trusteer", "Lua", "Fraud-detection", "low-code"]
date: "2025-03-16"
tags: [MacOS, software-development]
excerpt: "Bridging the gap between known and new fraud threats. Empowering client to rapidly respond to threats - without writing a single line of code."
---
Sophisticated, new emerging fraud attacks spread every few minutes, leaving customers vulnerable. Trusteer Pinpoint defends against known frauds, but responding to new threats can take days, even weeks. What if we could empower customers to block those threats in mere hours? That's the vision behind Policy Manager, a project I was the lead developer on the Pinpoint team. We've revolutionized fraud response, empowering customers with rapid, direct control

In early November, a select few of our customers were introduced to the new **Policy Manager** - allowing them, with a few clicks in _Trustboard_ (Trusteer's customer-facing toolkit), to create and publish custom rules that can override Pinpoint's risk assessment.

![policy manager visualization.jpeg]()
> Respond quickly to wide fraud attacks using a simple UI.

In turn, alerts (and associated automatic actions) on newly identified threats can be added as soon as they are recognized. Previously, an identified threat from a customer would require a lengthy process with different teams on Trusteer's side to analyze, identify, write and publish rules. During this process, our customers are exposed to a threat they know about. _Policy Manager_ enables Pinpoint to react quickly to new fraud threats, and our customers can protect their customers with as little friction as possible.

Not a few days passed from launch, and Policy Manager already started alerting on suspected frauds. A month later, a customer used _Policy Manager_ to stop a known attack that was previously blocked; a missed configuration caused the sessions to be detected in a different category than before, and the identified fraud case was not covered properly. The customer managed to respond and _**block it within three hours**_.

![[policy_manager_real_world_use.png]]
> Stopping a real fraud in the wild, a month from release.

Trusteer's Support team helped the customer set the rule in this case, but the UI allows the customer to add the rule themselves, reducing the response time even further with less friction.

### Fast, Friction-Free, Fraud Fighting

Policy Manager relies on a new UI in Trustboard that sends a specialized payload to the _Rules Engine_ component in Pinpoint, which turns it into Lua code reflecting the required logic. The code is synchronized to the different Pinpoint clusters using S3 within minutes. When Pinpont is called to provide a risk assessment via the **pinpoint_eval** endpoint, _Policy Manger_ runs right after Trusteer's _Policy_ (rules created by Trusteer's Security teams) The provided risk assessment will change according to the results from both components.

Trustboard's Policy Manager UI was crafted to be as simple as possible. An intuitive, no-code interface empowers fraud analysts to create custom rules without any programming knowledge. This allows them to focus on their core strength: identifying and responding to emerging threats with speed and precision.

The results and alerts produced by Policy Manager use the same channels as Pinpoint has already; customers (i.e. banks) do not need additional development to support or use it. The existing mechanisms the customer has for handling an alerted session will work just the same.

In _Rules Engine,_ the specialized payload is verified the JSON to meet a specific schema. It is then templated into Lua code and passes several correctness checks: the generated code needs to have a correct syntax and to be type-safe. Performance optimizations to the code are done during generation. Having the system generate the code allows us to understand what the _requirement_ is for the rule, and design efficient code around it.

![[Screenshot 2025-01-19 at 10.02.13.png]]
> A taste of the templating engine. Uses a [formal language](https://en.wikipedia.org/wiki/Formal_language) definition to create flexible code blocks.

_Policy Manager_ uses the same dataset as _Policy_, mitigating the need for additional database calls and avoiding a performance hit. It runs the code in a specialized sandbox environment, providing a very specific set of inputs and filtering for a specific set of outputs. When a rule triggers in _Policy Manager_, the output will reflect the new risk assessment, if relevant. Generating the code allows us, the developers, to iterate and optimize hot spots as they appear, and add additional abilities over time without breaking production code.

![[rules engine vs policy manager.jpeg]]
>Rules Engine in Pinpoint and Policy Manager in Trustboard work together to catch fraud, fast.

### One Engine To Rule Them

From a product stand point, **Policy Manager** is the shiny new frontend component, and **Rules Engine** is the backend one.

The _Rules Engine_ project, several steps from production use, laid mostly dormant in the last few years in Pinpoint's _Hub_ Service. It gained a renewed interest and an adaptation project into Pinpoint was together with the development of _Policy Manager_ (on the Trustboard side) mostly in Q3 2024. Both launched in Q4 for a select group of early adopters.

Originally, _Rules Engine_ had even more features: support for three tiers of rulesets - global, business and application; it had a dry (test) mode and metrics on which rule was used; rulesets can be crafted by activity name (i.e. login, transaction, etc.); custom list support and local variables, and more. Most of these features were stripped out of the Pinpoint adaptation, and some remain disabled in configuration. These are not forgotten but are waiting for a use case.

#### Rule Wrangling

The Trustboard team built a simplified and streamlined _No-code_ UI In parallel accessible to customers in Trustboard. The users can create rules in a _ruleset_; for each rule they can add and combine conditions, set the recommendation and score when conditions are met, and prioritize one rule over the other.

![[rule .png]]
> Creating a rule in Policy Manager's  UI.

The entire thing is done without exposing the user to any of the intricacies of the underlying code that's been run, and there are many - from variable types to parameter orders, null and error handling, its all neatly tucked away inside the templating engine and the sandbox.
![[ruleset.png]]
> Setting order, enabling and disabling rules in Policy Manager's UI.

At the moment, a single _ruleset_ is available to the customer, and publishing it will affect all of their applications at once. The UI builds the payload that is sent to Pinpoint upon a "publish" request, and _Rules Engine_ creates and distributes the code in minutes to the relevant clusters.

### What's Next?

Policy Manger already generates optimized code, deploys world wide in minutes, accessible to business users (also known as "non-programmers") and requires zero additional development from our customers. Where does it go from here?

Future planning includes near-immediate deployment at less cost (goodbye S3), support for checking more session attributes with additional functionality, dry/wet and tiered rulesets, list comparisons and much more.

### Thank you for reading!

#### Policy Manager is the work of many, among them:

- **Liran Tiebloom** (TPM)
- **Vikas Shankar Poojary** and **Anat Ilani** (QA)
- **Nati Braha, Gal Amuyal, Or Krichli** and **Yehuda Fireberger** (Trustboard)
- **Adi Meiman, Nir Nahum**, and I, **Amir Arbel** (Pinpoint)
