---
title: "Domain Categorisation - Finance ReactJS Site"
date: "2021-01-04"
categories: 
  - "miscellaneous"
---

I've wanted to play around with ReactJS to see what the fuss is about for a while now. I also had some ideas around finance domain categorisation and thought that i could nicely tie the two concepts together.

Some companies have different security controls around finance websites to avoid the complications with monitoring personal and sensitive information. In a similar vein to the [Python Packaging mirrors](https://akingscote.co.uk/posts/package-management-an-overlooked-security-vulnerability/), I figure i can publish my own site to get around this is ill founded and potential security risk. Often its not just company policy, but the tools themselves which support this process. For example, the [Symantec Bluecoat](https://sitereview.bluecoat.com/#/) is an extremely popular tool and its domain category list is commonly used in all kinds of firewall and egress tools to decide what policies to enforce. [SSL/TLS sandwiches](https://docs.fortinet.com/document/fortigate/6.2.0/new-features/213899/ssl-offload-sandwich-mode-6-2-1) may use domain lookups to decide what websites they can encrypt (for monitoring) within a corporate environment. If a corporate user accesses their personal bank account from a company machine, can the company legally snoop and monitor that connection? I believe that finance websites often have less strict security policies because of the often "grey" area regarding user privacy and rights, especially in our GDPR and privacy aware modern environment.

So with this theory in place, i decided to check my sites category.
![](/images/bluecoat-current-category.png).
Unfortunately this site is categorised as "Technology/Internet", but do they actually do their due diligence and can i just easily change my category?

![](/images/bluecoat-submit-for-review.png)
So i submitted a request to change the category.

But it turns out that they are pretty good and its not just some automated service mindlessly agreeing with requests.
![](/images/bluecoat-symantec-response.png)

So with this initial hurdle, i decided to try something sneaky. I went onto [https://www.expireddomains.net/](https://www.expireddomains.net/) to purchase a domain that is already categorised as `Finance`. I went for the obvious and just searched for a domain with `finance` in the name. ![](/images/financeadviceuk-expired.png)

Bingo! `financeadviceuk.com` seems to already be categorised as finance and was first on the scene in `2013`, which is long enough to be included in most crawlers. However, when i purchased the site, some analysers didnt recognise the site as finance, like the [zscaler url checker](https://tools.zscaler.com/category/).
![](/images/zscaler.png)

So what i decided to do is to actually use the domain to host a fake financial business, to try and show that anyone can be behind these sites. I have been meaning to have a little play with ReactJS, so this presents a good opportunity.

I can now present my lovely little fake site - [https://financeadviceuk.com](https://financeadviceuk.com)

![](/images/financeadviceuk-Finance-Advice-UK.png)
