# ASP.Net User Session Manager
After a user logs in to your web application (or a client authenticates to your API) you will usually want to maintain an *authenticated user session* so the user stays logged in for a while. 

ASP.Net provides good built-in support for Stateless Authenticated User Sessions. However, for security reasons, you may often want to use Stateful Authenticated User Sessions.  

*This library aims to provide an alternative set of middleware and tools to enable you to use Stateful Authenticated User Sessions with ASP.Net*

**NOTE: At this point in time, no code has been written. This page here is first and foremost intended as a Request For Comment to establish design criteria - and to confirm that there isn't already a solution out there.**

Please comment on the general ["Is this worth it" issue](https://github.com/NewOrbit/UserSession/issues/1) or raise a specific issue with comments. Thank you!


## The problem
Unfortunately there is a lot of confusion about the whole area of "how to keep users logged in", so the "problem description" here takes quite a lot of words to explain what the problem even is. Sorry.

After a user has *Authenticated* to your web application, you want to "keep them logged in" for a while so they don't have to keep re-entering their password or keep bouncing to Facebook or whichever other authentication method you use.
You would normally do this by issuing a *cookie* or a *token* which is then sent on each subsequent request and identifies the user as already having been *authenticated* to your system.  

"Keep them logged in" is referred to here as an *Authenticated User Session* or just *Session* for short. 
This is not to be confused with "Session State", which is an entirely separate concept (and something you'd usually want to avoid).  

There are many benefits to having an *Authenticated User Session*; If you didn't, you'd have to send a username/password (or something similar) with each request - and validate it, which is usually deliberately slow.  
There are some scenarios where you may do this with an API client that makes infrequent calls, but that is a separate discussion.

### There are fundamentally two types of *Authenticated User Sessions*;
- **Stateful**, where the server keeps a list of Authenticated User Sessions and the client just has a Session ID.
- **Stateless** where the client holds some "state" to identify them as logged in and, usually, some additional data about them such as name, roles etc. This approach uses various cryptographic mechanisms to make sure the user can't falsify this information.

You can use either cookies or tokens with either approach, for example a cookie can contain either a session ID or a full set of user claims, as can a token.

### Weighing up the options
There are pros and cons to both approaches and the right approach for a given system will depend on a trade-off;
- Stateless is inherently more *scalable*; Stateful requires a server-side session store with shared access by all front-end servers, which can be a bottleneck.
- Stateful is probably more *secure*; In the stateless scenario there is no way to invalidate a session from the server; if an attacker somehow steals a user's session token or cookie, they can impersonate that user and there is very little you can do server-side to stop them; even the legitimate user logging out or changing their password will have no effect. There are a few mitigations, but they are likely to negate the performance benefit of being stateless. The [OWASP guidance](https://www.owasp.org/index.php/Session_Management_Cheat_Sheet) is clear that it must be possible to invalidate user sessions server-side.
- Stateless combined with tokens and a requirement to log a user out after a period of inactivity requires some fairly clever javascript code to be written by you because tokens, unlike cookies, can't be automatically refreshed by the server. And no, OIDC Refresh Tokens are not the answer for Web applications. Cookies don't have this problem, though.

These things are all relative; how big does your system have to be before the scalability of stateful sessions becomes a problem if you use things like Redis? How likely is it that an attacker can steal a session cookie or token, given the protections that you can put in place? If you just have the one system to build and maintain, do you really mind spending a bit of extra time to write the code that refreshes the token in your front-end JS application?

One thing you may not have much control over is external security audits; In penetration tests during 2016 I have seen a big increase in testers insisting on stateful sessions - whether it is really warranted or not.

### Implementation options
The default mechanisms in ASP.Net are generally *Stateless*, issuing either a cookie or a JWT token with a full claims principal in it. 

*The aim of **this** library is to supply alternative cookie and token middleware that supports Stateful Authenticated User Sessions*

You should be able to 
- Use the library to create Sessions, which are stored in a server-side sesion store.
- Use library middleware to validate sessions on each request and populate the Claims Principal on each request with data from the session store.
- Use library-provided sessions stores to store sessions in Redis, DocumentDb, SQl - maybe even Azure Table Storage - or provide your own.
- You may also use this for pure API functions, though for those scenarios you should also consider oAuth or OIDC as probably better options.

### But doesn't OIDC/oAuth/Identity Server/Facebook/Google/Azure AD B2C/etc solve all this?
Unfortunately, with very few exceptions, those Identity Providers only deal with the *Authentication* part; The user turns up at your web application with, say, a token from Facebook. That token is just there to say that Facebook knows who this person is. It is then up to you to read that information and set up your own User Session by setting a cookie or issuing your *own* token to the client.

As a matter of fact, the use of an external Identiy Provider often blinds developers to the fact that they are still responsible for the User Session, sometimes causing significant security problems to be introduced.

One exception is Azure AD B2B Authentication, which in certain setups will actually handle the user session for you as well.

**TODO**: Investigate how much IdSrv 4 provides! v3 didn't support this fully, but there is a chance v4 does!

It is *possible* to build session management on top of Identity Server; basically you have a session store in IdSrv and you use custom middleware to call back to IdSrv on each request to validate the session and obtain the claims for the user. This works, but it is a fair bit of work and probably only right for certain specific scenarios. 

**TODO**: There is a (small) chance that ASP.Net Core has this built in, though I doubt it from history. Even if it does, there is still the issue of older systems - but it would make the case for this library a lot less compelling.

### Notes
- It is arguably misleading to talk about Stateful and Stateless Authenticated User Sessions as there is state in both, the difference is only in whether the server or the client keeps the state.
- There is a lot of confusion about this on the internet, including a series of articles that equate cookies with stateful and tokens with stateless. That is, of course, completely wrong; you can use both cookies or tokens with both stateless and stateful implementations.
- Technically, you probably wouldn't wrap a simple Session ID in a full JWT token, you would probably just use the Session ID on it's own, but it doesn't really make much difference. The term *token* is used here to mean the scenario where the server passes a string value to the client, which the client then stores somehow and transmits to the server with each request. This is a bit lazy TBH as we are just taking about sending a plain Session ID, not a proper token with lots of data in it. Please forgive and feel free to suggest a better term.
- ASP.Net OWIN Cookie authentication middleware does have a (thinly documented) ability to store the claims principal in a server-side store and just send a Session ID to the client. However, this is mainly concerned with reducing the payload and does not have the additional security features being discussed here.
- Up until early 2016, the prevalent advice was that JWT Tokens were safer than Cookies for authentication and, indeed, penetration tests would reflect that. However, that advice turned completely on it's head in 2016 when penetration tests started to complain about the use of tokens over cookies. The reason, in short, is that - as imperfect as cookies are - we have spent many years building protections into browsers, including Secure-Only, Http-Only and so forth. No such protection exists for Tokens, meaning they actually have a much bigger potential attack surface. This is not directly relevant to this library, but is closely related so is worth mentioning. Unfortunately, the debate is highly charged and is muddled by misunderstandings, the massively overloaded term "Token" and the conflation of Stateful/Stateless with Cookie/Token.

## Design goals
The User Session Manager will be designed to be called *after* the user has been authenticated, whether locally or by some external Identity Provider. It will establish a User Session, which is stored in a server-side session store, and issue a cookie or a token (other options can be introduced). It will also include the appropriate logic to validate the cookie/token on each subsequent request and populate the Claims Principal for the request.
The library must;
- Be fast and add minimal overhead to each request, both in time and CPU.
- Must have support for "Remember me"
- Must support for both ASP.Net Core and older ASP.Net, the latter probably through OWIN.
- Be able to integrate with the existing auth middleware, i.e. there are mechanisms for, say, Facebook authentication middleware to hand off to the cookie middleware to set an Authenticated User Session Cookie; the middleware in this library should be able to be used as a replacement for said cookie middleware, if desired.

## Solution design
It is worth reading the [OWASP session guidance](https://www.owasp.org/index.php/Session_Management_Cheat_Sheet), which covers a number of the items to be considered.

### Session ID
The session ID that will be handed to the client web browser is simply a cryptographically generated, random string of an appropriate length. Current suggestion is 40 characters to ensure a very high level of entropy.

There is no point in encrypting or HMACing the session ID; encryption will just change the string to another string of a given length and HMAC will just make the random string a bit longer; From the point of view of brute-force attacks, neither encryption nor HMAC will provide any additional security that can't be achieved just as well by simply making the Session ID longer. Furthermore, both encryption and HMAC would introduce a performance hit on each request.

### Storage of Session IDs and user data server-side
If we store the raw session IDs server-side then this may cause a problem if an attacker gains access to the session store. It is therefore suggested that we only store the Hash of the Session ID server-side. This does introduce a small performance hit on each request, but using something like SHA1 this will still be very fast, whilst being secure enough to at least hinder an attacker. 

On the server, we will store a list of Claims in the Session Store, meaning we can take a Session ID from a user and create a Claims Principal if desired.

The intention is to provide a Redis, a SQL and a DocumentDb Session Store initially, with the ability to add more.

### Idle timeout
Cookies have a decent mechanism for handling idle timeout, but the same cannot be said for tokens where a lot of client-side work is required to keep track of token expiry and renewal.

It is much simpler for the idle timeout to be enforced server-side; the Session Store must keep track of when the User Session was last accessed and automatically invalidate expired sessions.

### Cookie lifetime
A cookie is given a lifetime when it is issued. One option is to tie it to the browser session; the cookie will persist until the browser is closed. The benefit of using this is that the user is effectively logged out when they close their browser. It does mean that the cookie will hang around for a long time if the user does not close their browser - but the session will be invalidated on the server anyway.

Alternatively, a cookie can be given a specific lifetime and then renewed once in a while to implement an idle timeout on the client side. Apart from added complexity, the main problem with this is that the cookie can now persist even when the browser is closed, arguably making this less secure. 

It is suggested that the "browser session cookie" approach is used. 

### Multiple concurrent user Sessions
A user can log in from multiple devices, each log in will be a separate session but tied to the same user. This information will be readily accessible in order to;
- Allow a user to see all their active sessions
- Allow a user to "log out from everywhere"
- Allow the system to kill all user sessions when the user changes their password, for example.

### Binding the Session ID to Other User Properties
Session hijacking remains a distint possibility; if an attacker manages to steal your cookie or token or somehow identifies your session ID they can impersonate you. If you are aware of this you can just log out and the session will be invalidated server-side - but what if you are not aware of it?  
As an additional precaution, the library should be able to store additional data, such as the user's IP address and browser user-agent, as well as potentially more information going forward. These are far from fool-proof and the IP address in particular may prove tricky, but it adds another layer of protection The suggestion would be to automatically invalidate a session if any of the User Properties changes. This would force the legitimate user to login again, but at least it also blocks the attacker.

### Logout, password change etc
When a user logs out from the application, their server-side session should be invalidated. This ensures that even if an attacker obtains a cookie or a token, they will not be able to use that after the user has logged out.

On password change, we would recommend that the application invalidates all active user sessions as well as all Remember Me sessions (see below).

### Periodic regeneration of the Session ID
*Open Question*  
If desired, we could make it so a different session ID is generated periodically whilst a user remains logged in. This is relatvively easy to implement with cookies, as it is easy to just issue a new cookie (though the system would need to allow for a small overlap between the two sessions to handle concurrent requests). However, it would be much more difficult to implement when tokens are used as there is no easy way to exchange those.

### Intrusion detection
*Open question*  
It would be possible to build in some mechanisms to monitor levels of invalid tokens being submitted, possibly from specific locations/IP addresses etc and maybe raise alarms or dynamically block certain IP addresses temporarily.
You could argue that this is the responsibility of the application - but to do this kind of monitoring requires shared state between servers, and this library is in a unique position as it has access to exactly that.

### Remember Me
Users often request a "remember me" or "keep me logged in for two weeks" feature. Whether or not you want to support that from a security point of view is for you to judge against your particular system. However, this library will have support for it, should you want it.  

Standard ASP.Net will achieve this by simply giving you a session cookie that is very long lived and it will usually auto-renew.
This library will take a different approach;
If a user requests "to be remembered", the system will create a record in a *separate* store and will issue a cookie with the remember-me ID.  
When a user is not authenticated and the remember-me middleware is enabled and a remember-me cookie is present, the library will interact with the application to establish a new user session.  
This approach has a number of advantages, including;
- Remember Me can have a *fixed* duration, for example 2 weeks, after which the user will have to re-login regardless.
- When a user is automatically re-authenticated, a new session is created which uses any updated user information, roles etc.
- When the user chooses "logout" or "logout from everywhere" their "local" remember me or *all* remember-me cookies respectively can be invalidated.
- On events like "change password", the system can automatically invalidate all "remember me" sessions.

