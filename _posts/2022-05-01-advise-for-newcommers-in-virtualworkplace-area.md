---
layout: post
title: "Advise for newcommers in VirtualWorkplace area"
permalink: "/advise-for-newcommers-in-virtualworkplace-area/"
subtitle: "Virtual Workplace aka EUC is not always a bread and butter"
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [ReadmeFirst ,HomeLab ,Citrix ,CVAD ,RDS]
categories: [RedmeFirst ,HomeLab ,Citrix ,CVAD ,RDS]
---
Those advises are not mine, they come by the End User Computing Veteran, who is spreading and sharing his knowledge, in the community a well known person (hat's off) Carl Webster, who created his very well detailed guide [Building Webster's lab V2.pdf](https://carlwebster.com/building-websters-lab-v2/) - page 1312 of his guide for the EUC lab builders. He has spent 706hours creating this guide.

Here comes the quote from his guide:

## Advice
1. Always learn.
2. Always improve yourself.
3. Be a better person today than you were yesterday.
4. Learn good habits in the lab. What you do in practice becomes what you do in real life.
5. Learn to document your lab. Documenting your work and configurations is an excellent habit to learn. I have never known anyone who complained about having too much documentation.
6. Learn to document the changes you make in the lab. While you may not need a formal change control process for your lab, learning to document changes BEFORE you make them is an excellent habit to learn. Here are some basic examples of a change control document.<br>
a. Reason for the change<br>
b. What is wrong in the current environment that requires the change<br>
c. What is the change<br>
d. What is the impact of the change (i.e., all servers require a restart)<br>
e. Validation of change<br>
f. Is there a rollback plan<br>
7. Do not practice stuff in the lab until you get it right once. Practice until you cannot possibly get it wrong.
8. Remember your physical health. We I.T. people have a reputation for sitting on our rear ends all day long. Get up, get some exercise, even if it is just walking up and down the stairs. Move.
9. Remember your physical health #2. We I.T. people have a reputation for eating junk food and rarely eating much, if any, healthy food.
10. Remember your mental health. We I.T. people have a reputation as loners and loving isolation. Have friends. Spend time with your friends. TALK to your friends.
11. Remember your mental health #2. We I.T. people have stressful jobs. Keeping stress bottled up inside can lead to bad heart health and bad brain health. It is OK not to be or feel OK. TALK to someone.
12. Have a hobby that does not involve computers or technology. Take a mental break from computers and technology.
13. Don't be afraid to make mistakes. Preferably make them in your lab and not on the job or at a customer site.
14. Learn to automate. Now that you know the basics of creating hosts, servers, virtual machines, virtual networks, virtual storage, and much more, you do not want to do this manually at scale.
15. Automation comes in many flavors. It would be best to read about software like Ansible, Chef, Jenkins, Puppet, and others. Many vendors offer commercial automation software—for example, Microsoft System Center Configuration Manager and Microsoft Deployment Toolkit.
16. Cloud is the future but is not cheap to use in a lab unless you become a Microsoft Most Valuable Professional (MVP) or get access to credits for Azure, AWS, Google Cloud, and other Cloud providers.
17. Start your lab small. You don't need to go into thousands of Dollars/Euros/Pounds/Rupees of debt to build a lab. My first lab was an old computer built from many other computer parts thrown away by friends and customers. Back in 1984, it was amazing what you could do in 1 MB of RAM, a 30 MB hard drive, and a 12" monitor at an incredible 80 characters by 25 lines display.
18. As we did in this PDF, always think about security. That is why I left the Windows Firewall and Windows Defender enabled on every computer. Learn about TCP/UDP, Ports, Inbound and Outbound network traffic.
19. As we did in this PDF, always do one thing at a time. Take your time, Be patient. If you make five changes at once and then test and the test fails, which of the five changes (or combination of the changes) caused the failure?
20. There is no one right way to do anything in technology. This is the old cliché of asking 25 consultants how to automate a process or design a Citrix/Microsoft/VMware infrastructure, and you receive 25 different proposals/viewpoints/opinions. And the 25 consultants tell you why the other 24 consultants are wrong or idiots.
21. Don't neglect people skills. Soft skills are what helps you succeed in any career. Check out this video by the fine folks at Thrive-IT. Elevating Your Soft Skills.
22. I asked my fellow CTPs and vExperts what advice they would like to offer you.<br><br>

*CTPs:*<br>
**Benjamin Crill:**
Always be willing to TRY a solution. You don't know that it is bad/won't work/not what you want/etc until you actually try something.

**Manuel Winkel:**
Work in the test environment according to the approach - trial and error - everything is possible and can be tested if it is not productive ^^

**Guy Leech:**
If you've got RAID arrays for some resilience against disk failure, have a hot spare where possible, and if not, at least set up practice alerting so you get notified on a disk failure. Make sure you size a UPS correctly, although mine mainly protects against occasional Residual Current Device (RCD) trips, so it only needs to provide a few minutes of power. (Webster. Check the detailed explanation of RCD at [Site issues with Earth leakage](https://www.apc.com/us/en/faqs/FA156793/).)

**Julian Mooren:**
1. Do small steps. Especially when automating because it can get quite overwhelming
2. Maybe build the lab with a good friend or colleague. Together is better.
3. Invest time in yourself = Better Skills and career opportunities
4. You don't need to have a 4000$ Home lab setup. Set yourself a budget and start with a tiny machine and build component by component. If you are still enjoying it after some months upgrade.
5. Watch your Credit Card bill when building your lab on Azure or another public cloud.
6. Don't be afraid to try/learn something completely new. Play with the solution and make yourself an impression of it. Even if it is a product from the competitor

**Mads B. Petersen:**
Controlling access to management, think jump host, MFA for windows logon and such, which I see many companies don't do very well.

**Leee Jeffries:**
Invest in yourself, get some hardware to play around with, work on the fundamentals, and focus on what interests you. You'll be surprised about how quickly you learn when you're keen to get started. Ask questions to your peers in the community; they are all friendly folks. Lastly, share what you know.<br><br>

*vExperts:*<br>
**Stephen Jesse:**
"don't be afraid to experiment," meaning do dumb stuff here not in production, so you know why you don't do it

**Pawel Kubik:**
If I may, I would add just one thing, Always try to keep it simple.

**Mike Martino:**
Aim to be 1% better every day - small chunks of effort adds up. Especially working on things in your home lab and/or when learning and studying for a cert when there can be so much to do that you need to focus on little bits at a time to get through it.<br><br>

## Conclusions
No one is perfect.
Don't be afraid to start your own blog and document your learning experiences. If you have questions, someone else is asking the same questions. When you figure it out, document what you learned, the learning process, and the lessons learned along the way (also known as, oops, I made a mistake, and I
don't want you to make the same mistake.).
Learning something new can be stressful, challenging, and frustrating. But it is rewarding when you figure the new stuff out and feel that sense of accomplishment.

Last edit: 2022.05.01
