---
layout: post
title:  "Halloween horror with AWS Savings Plans"
tags: ["aws", "costs"]
header: "/img/SavingsPlansPricing.png"
comments: true
---

# A story about how purchasing savings plans can go badly wrong

Recently we decided that it was time to purchase our first AWS Compute Savings Plan. 

Our work streams had settled and we had a good prediction of our EC2 workload required for the next 12 months.

As per the AWS best practices recommendation we have implemented AWS Organisations with many linked accounts for our different workstreams. 

Each workstream generally has a testing account and a production account, where the test account usually has a developer support plan and the production account has the business support plan applied. 

We decided to begin with our developer tooling account which housed our Gitlab runners.

We did our due diligence on our research for RI vrs Compute and EC2 Savings Plans.

We put our findings forward to finance and they agreed, giving us the go ahead to purchase a combination of Compute and EC2 Savings Plans for the best return on our predictions. 

1 year plans were purchased all upfront with expected savings of about 23% or $1500 a month. 

We selected Savings plans due to their flexibility to be redistributed among the other linked accounts in the case that they weren't fully utilized. 

We wanted to ensure that the savings plans were applied to the developer tooling account first before excess was distributed to the other accounts so we purchased
the savings plans in the tooling-production account which our gitlab runners execute. 

Our total cost of savings plans paid for upfront was just a little shy of $23k. 

Now...this is where our horror began. If you recall, we had Business Support plans on our production accounts. 

The pricing for Buiness support plans is starts at 10% for the first $10k spent in the account. 

![Support Plans Pricing](/img/SavingsPlansPricing.png)

Tacking on a bill of $23k to this account meant that our bill for the Support Plan immediately jumped by almost $2k and cancelled the savings we made for almost 2 months. 

I guess the morale of the story is to be aware...very aware of where and how you purchase your savings plans. Purchasing these savings plans in another account which didn't have the Business support plan would have saved us a pretty penny. 