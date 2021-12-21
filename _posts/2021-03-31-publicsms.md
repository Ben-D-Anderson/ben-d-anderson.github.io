---
layout: post
title: PublicSMS
description: Anonymous sms conversation using public temporary phone numbers
readtime: 1
tags: project
---

[GitHub](https://github.com/Ben-D-Anderson/PublicSMS)

PublicSMS is a program which scrapes implemented sites for public temporary phone numbers and uses them to receive text messages. It also uses implemented sending methods to spoof messages from that number so that a target phone can have a fully functional conversation with the public temporary phone number without the user of the application using their real phone number.

Features:
- Send and receives SMS messages without using your own phone number.
- Add your own implementations of websites to get public temporary phone numbers from or services to send sms messages from.

However, there are pitfalls to the approach taken with this project:
- All text messages sent by a target phone to the public temporary phone are publicly accessible and readable, including the number of the target phone.
- Implementation from the websites it scrapes are subject to change at any time so may not always be functional.
- To spoof messages from the public temporary phone number, the program currently uses a service called Clockwork-SMS (renamed TextAnywhere) which requires an account, api key and credit - each text costs approximately £0.055 to send. This service is not necessarily anonymous as they do not support anonymous payment methods.

This project functioned as a nice experiment but usage of the program is highly discouraged due to the flawed design mentioned above. I've ultimately decided it's not possible to achieve anonymous sms communication using the method I had in mind, however this project will remain publicly visible as a proof-of-concept idea.
