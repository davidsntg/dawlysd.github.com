---
layout: post
title:  "NSGaC - Network Security Groups as Code"
author: davidsantiago
categories: [ network ]
image: assets/images/nsgac.png

---

[NSGaC](https://github.com/dawlysd/azure-nsgac) is an open source project that allows  to **manage Network Security Groups** (NSGs) rules for VNets in Azure, **as code**.

NSGaC permits:
* **Auditability**: know who created a rule, when and why
* **Reduce implementation lead time**: once a rule has been requested (via pull request), if all the validations (terraform syntax, manual/automatic approval) are successful, the rule automatically goes into production. The rollback is also part of the CI/CD if needed.
* **Maintain consistent rules**: anything manually created in the portal is deleted. The only source of truth are the nsg rules in this repository.
* **Integration with ITSM systems**: CI/CD is extensible to fit organizations ITIL processes.
* **Enforce micro segmentation easily**: create isolation and determine if two endpoints should access each other by managing rules correctly.

Checkout [GitHub repository](https://github.com/dawlysd/azure-nsgac) to start using NSGaC!
