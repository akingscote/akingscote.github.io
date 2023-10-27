---
title: "What is Private Data? (1/3)"
date: "2020-06-12"
categories: 
  - "miscellaneous"
  - "play"
  - "security"
tags: 
  - "data"
  - "open"
  - "privacy"
  - "security"
coverImage: "shield-1086703_640.png"
---

I found myself recently wondering, what exactly is private data? What i mean is, how much information about me is too much for a stranger to know. Of course there is Personally Identifiable Information (PII) which is any data that could specify an individual, but thats not really what i mean. Im interested in what information is sensitive for me to give away, not just something that can identify me.

This project came about when I noticed a friend of mine was tearing their parcel label off of their package that they had received from Amazon. I asked why they were doing that and they replied "i dont want people knowing where i live". My initial thought was that mindset is naive - surely if "people" want to figure out where you live, it wouldnt be that hard? So I got thinking, is your address personal information? My friend obviously seems to think so - he dosent want people knowing where he lives. So where do you draw the line? Full Name? Address? Email Address? Phone Number? Job? Salary? What car you drive? Financial info?

Nowadays, most people have a digital footprint. People are certainly getting wiser to their online privacy, but definately not everyone. A lot of people simply dont care and why should they? As long as you dont become a victim of identity theft and if you have nothing to hide, then whats the harm? The obvious answer is that you should be as conservative as possible about what information you leave accessible - at least thats what most cyber professionals would say. Personally, im not so sure. We live in the information age, the whole world is data driven. There are amazing progresses being made every day and i quite like the idea of a digital footprint. Something for me to leave behind, an insight into the lives of people that has never been available before. However there is a limit and I dread to think what information is gather about me unknowingly.

![](/images/google-timeline.png)
The above screenshot shows my google timeline from 5 years ago. It shows where i went that day, pretty much to the minute. That information is secure in my google account (at least i hope) and is not publicly accessible. Whats weird is that is just one department of one company gathering information on me, let alone the hundreds of systems i inadvertently interact with every day. Personally, i really like this feature which is why i leave it on. Im sure there are plently of people out there that hate the idea of being tracked. Ive also noticed google and facebook analysing my photos and automatically tagging peole i know.

So what information do I consider sensitive? Obviously my passwords, but personally I think thats about it. If I really wanted to be private, I would not have a smart phone, social media, email, mobile banking or any online prescense at all. I think as soon as you sign up to the information age, you give up your privacy. Unfortunately many people dont realise this and perhaps share too much.

I think that my daily movements is sensitive information. I think that my salary is sensitive information. I think that my address, my email address, my job, my name cannot really be sensitive information. Personally, thats my demarkation line. I order items online, so whoever im ordering from will have my address. It could be a sketchy warehouse in China with their government compiling a database or there could be a huge postal service conspiracy that stores everthing that goes through it. If you send out a CV, does it not contain a lot of sensitive information (work history, education, address, mobile number) that you send to unknown "recruiters". The second i order things online, im giving my address to people I dont know. So i really do think that ripping your address off a parcel is a bit silly. If a somebody was going through your bin, then they are likely already at your house and would know your address. Or they are at a tip and can determine that you ordered _something_ from Amazon - so what? I dont think thats particularly sensitive information (at least for me). However, I do also think that this is circumstantial and depends on individuals. For example, what if you were fleeing abuse, or a teacher or had a unfavourable job? You certainly wouldnt want your address being plastered online, nor your email.

So if your name and address isnt really that sensitive (for the majority of people), why isnt it easily publicaly available? Why cant I view who lives in a house when i zoom in on a property? Something that I dont think is being done, is trying to see how much information can be gathered by **cross referencing** multiple sources of data. One source on its own might not be frightening, but combining with numerous other sources really could lead to more information than we are comfortable with being shared.

So harnessing my newly found knowledge of Tegola and GIS, i thought i would try and piece together some information using public and open data sources and plot them on a map of my local area. There are infinite data sources out there, but im going to focus just on what i can ethically mine. Ive quickly realised that in order to get into the really juicy datasets, companies (facebook, linkedin, twitter) have put up paywalls to capitalise on their APIs. Unfortunately, i dont have the budget to tap into these behmouths which is such a shame as i think it could provide some valuable (and scary) results. If I had say £3K of funding, i think I would be able to gather information on thousands of people in my area.

So what sources are public, open and ethical? Im after information in Reading Berkshire where I live. To start with, the Open Register lists the names and addresses of individuals who are registered to vote and agree to be put onto the Open Register. The government/council keep a closed register with everyones name and address listed which is used by the council and authorities.

The open register is an extract of the electoral register, but is not used for elections. It can be bought by **any** person, company or organisation. For example, it is used by businesses and charities to confirm name and address details. The personal data in the register must always be processed in line with data-protection legislation. [http://www.legislation.gov.uk/en/uksi/2013/3198/schedule/3/chapter/2/made](http://www.legislation.gov.uk/en/uksi/2013/3198/schedule/3/chapter/2/made)

The open register is my first place to go. The official legislation says that it can be brought by anyone. However, on the Reading council website they say "only a company" may purchase a copy. After some persuasion that legally they cant deny me access, i was sent over a copy after paying an administration fee. ![](/images/openreg-1.png)

I was sent the information for the whole of Reading and using Golang, i opened each file and processed them into a single CSV. In the end, I got 38,742 entries of individuals and their addresses from the Open Register - so about 20% of the population of the town.

The Reading council also list their publically available registers - [https://www.reading.gov.uk/publicregisters](https://www.reading.gov.uk/publicregisters) One of the most interesting data sources is the Houses of Multiple Occupancy (HMO) - [https://www.reading.gov.uk/media/3768/HMO-register/pdf/HMO\_REG\_11032020.pdf](https://www.reading.gov.uk/media/3768/HMO-register/pdf/HMO_REG_11032020.pdf) Meaning, if your name is on there you are a legal landlord and own the property which is listed that you are renting out to multiple tenants.

As well as this, the governement also list open data sources - [https://data.gov.uk/](https://data.gov.uk/) Already, there are thousands of potential sources that can be used, without even trying to touch the social media platforms.

I'll pause this post here as i think this is a nice introduction into my project. The next two posts will show some of the technical details on what ive been doing.
