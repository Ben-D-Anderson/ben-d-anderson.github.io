---
layout: post
title: Web Security Basics - Cookies & Authenticated Sessions
description: Here we discuss the basics regarding how websites store a user session and keep you logged in - with a focus on ensuring security.
readtime: 6 minute
tags: cybersecurity, programming
---

## Sessions
Websites can serve lots of users at any one time; they need a way to recognise which user they are currently interacting with - this is where sessions come in. When you sign in to a website, all future interactions you have with the site need to be associated with your user account. Without the concept of 'sessions', every request you made to the site would seem completely unrelated and would not be associated with a user account in any way. Therefore, in order to keep you logged in for all the actions you carry out, a website needs to have some way to store data in the your session, such as which user you are logged in as - enter cookies.

## Cookies
Cookies are one way in which websites keep a user logged in throughout a session. Web browsers allow websites to create and read cookies (belonging to that site) from a users computer, all a user's cookies for an individual website are actually attached to every single request the user makes to that site. We won't go into specifics regarding how this works, but the important thing to understand is that for every request you make, the website reads the cookies that are sent along with it; the website might then choose to do something with these cookies, or it might not.

One of the most common use cases of cookies across the web, is using them to check that a web request came from an authenticated user. Another significant thing to understand is that a cookie is effectively a key-value pair, a cookie has a name and a value.

## Changing your cookies
Because of the fact that cookies are stored client-side (on the user's computer), the user can control the values of all cookies on their machine. You can view and change them on this website right now, just do the following:
- Right click the page
- Select `Inspect`
- In the opened panel, select the following tab depending on your browser:
	- `Application` if you are on Chrome or Microsoft Edge
	- `Storage` if you are on Firefox
- On the sidebar, select `Cookies` and click on `www.benanderson.xyz`

Below is a visual to demonstrate this process on Microsoft Edge:

<img src="/assets/img/check-cookie-demo.gif" />

You'll then see all the cookies used by this website - which you can edit by clicking on them, changing the value and refreshing the page.

## Insecure authentication with cookies
Let's imagine you login to a website with the username `user1`, the site could create a cookie on your computer called `account` (could be named anything) with the value `user1`. Now everytime you make a request to the site, the website will do the following:
1. Read your cookies
2. See that there is a cookie called `account`
3. See that the value of the `account` cookie is `user1`
4. Accept that you are logged in as a user with the username `user1`
5. Display the relevant content for your account

However, this is a very insecure way to save the user's account to a cookie due to the fact that a user can change their cookies. User A could simply change the value of the `account` cookie stored on their machine to be user B's username, then user A would have full access to user B's account.

A few developers may believe a solution to the problem posed above is to simply attach the user's password to the cookie and confirm their credentials are correct on the website's backend. Whilst this seemingly works on the surface, it opens a plethera of other potential vulnerabilities that need to be closely analysed as to their potential impact on both the website itself and the end-user. For example, if there was a cross-site scripting (XSS) vulnerability elsewhere on the website and the cookie wasn't properly secured, then a malicious actor could steal the credentials of all the site's users. Another issue to consider is the possiblity of the user's device becoming compromised, their password will instantly belong to the actor that compromised their system as cookies are stored on the user's machine in plain-text.

Due to aforementioned reasons, and others, attaching a password to the cookie is not a recommended solution - and certainly not accepted as best practice.

## Secure authentication with cookies
There are two main approaches to secure authentication using cookies.
- Using a unique session token
- Using a JSON Web Token ([JWT](https://jwt.io/))

### Storing a session token
We can say that every session a visitor creates when navigating to our website, can be assigned a long, randomly generated token/identifier - often referred to as a 'session token'. These would be long enough and random enough to the point where it would be effectively impossible to guess another user's session token. This unique session token can be stored as a cookie on the user's machine, and now when they login to our website, we associate their session token with their user account on our website backend - usually using a database.

From this point on, for every request we receive with a session token cookie attached, we check which user account that session token is associated with (if any) on our website backend. If we find an associated account, we can accept that the user who made the request must be authenticated as the associated account.

The only challenge we have left is that if a user's session token is stolen in any way, their account will be permanently compromised. Our only real solution for this is to give our session tokens expiry times. You shouldn't try delete the cookie from the user's browser as that will yield little success, and the session token will still be valid and could be used by a malicious or non-malicious actor. Instead, the session token expiry time should be stored in the database used by the website backend, and any account associated with a session token should be disassociated from it after the session token has expired.

The session token's expiry time could be an arbritrary amount of time, it heavily depends on how long you expect an authenticated user to be interacting with your site before they leave it, however typically most websites won't have a session token valid for longer than a day.

### JSON Web Tokens (JWTs)
JWTs are a much different approach than your classic session token model and only really came into fashion in the late 2010s. Just as before, the token can be stored as a cookie (although typically they are not) and read by the server, however the JWT is effectively a bunch of information with a cryptographic signature appended to the end to ensure it hasn't been tampered with. This stops the user from being able to edit the token, because as soon as they do the cryptographic signature at the bottom of the token no longer suits the data above it - the JWT becomes invalid. Furthermore, the signature can't just be changed, due to the fact it makes use of [hashing](https://en.wikipedia.org/wiki/Hash_function) and a secret key which only the webserver knows, this makes it so that only the website is able to issue and change JWTs.

The JWT approach is beneficial in the sense that you don't need to store anything in a database to make them work, they can be validated purely by looking at the JWT and checking that the signature matches what is expected due to the data. If you then want to add the same expiry feature as with session tokens, you can just add an expiry time to the JWT and everytime you check the validity of JWT you now also need to ensure that the expiry time on the JWT has not already passed.

The only thing with JWTs which are often considered a weakness, is they can be very difficult to invalidate; if, for example, a user account is deleted by an administrator, the user's JWT doesn't just stop being valid, it remains valid until its expiry time is reached. This could lead to unexpected behavior when the website tries to execute actions based on a non-existant user account.
