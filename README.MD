# ASP.Net User Session Manager
After a user logs in to your web application (or a client authenticates to your API) you will usually want to maintain a *user session* so the user stays logged in for a while. You may do this by setting a cookie or issuing a token.  
However, the built-in methods in the various versions of ASP.Net (incl. Core) for doing this are somewhat minimalistic. 

*This library aims to provide a more comprehensive solution for user session management that complies with security best practices.* It is intended for use in web applications, including SPAs and hybrid mobile applications where the client cannot be trusted. It will be designed to also be usable from rich clients, including server-to-server use, though for those scenarios there are more robust and arguably better solutions available in protocols like OIDC and oAuth.

**NOTE: At this point in time, no code has been written. This page here is first and foremost intended as a Request For Comment to establish design criteria - and to confirm that there isn't already a solution out there.**

**Terminology**  
The term *token* is used here to mean the scenario where the server passes a string value to the client, which the client then stores somehow and transmits to the server with each request. This is a bit lazy TBH as we are just taking about sending a plain Session ID, not a proper token with lots of data in it. Please forgive and feel free to suggest a better term.

## The problem
We are used to talking about Authentication and Authorisation but, unfortunately, the concept of the User Session is obfuscated and often confused with "ASP.Net Sessions", which are usually a bad idea. For example, there is a "cookie authentication middleware". This middleware is (marginally) involved in authentication but is actually primarily about handling the User Session, i.e. checking the cookie on each request and populating the Principal on the request. It should arguable be called "cookie *session* middleware".

In addition to being somewhat confusing, the built-in options are also somewhat deficient in the security stakes. A standard ASP.Net session cookie is self-contained and there is no way to invalidate it on the server, which is contrary to [OWASP guidance](https://www.owasp.org/index.php/Session_Management_Cheat_Sheet) because it makes certain attacks more likely. I should mention that there is an option for ASP.Net cookies to just contain a session ID and store the rest of the information server-side, but the documentation is sparse and it lacks certain helpful features. 

Tokens that your application issue have essentially the same problems as cookies *plus* there is no simple way to have an "idle timeout" on a token (Refresh Tokens are not applicable in web scenarios, they are for secure clients).

### Stateless cookies/tokens vs server-side user sessions
Most ASP.Net built-in options and indeed many recommended approaches for User Session management is to essentially store all the relevant information (Claims or otherwise) in a Cookie or a JWT Token. It saves you from having any server-side storage of sessions and certainly confers a scalability benefit, at the cost of a little extra bandwidth.  
For many scenarios this approach is fine, but is quite lacking in a security context; if an attacker gets a copy of a user's valid cookie or token there is now no way to invalidate that. It doesn't matter if the user logs out or changes their password, the attacker is still in there.  

There is a tradeoff here between scalability and security and you need to decide what is right for you. I do know that in pen tests during 2016, the lack of server-side sessions has gone from advisory to must-fix in the ones I see. 




### But doesn't OIDC/oAuth/Identity Server/Facebook/Google/Azure AD B2C/etc solve all this?
Unfortunately, with very few exceptions, those Identity Providers only deal with the *Authentication* part; The user turns up at your web application with, say, a token from Facebook. That token is just there to say that Facebook knows who this person is. It is then up to you to read that information and set up your own User Session by setting a cookie or issuing your *own* token to the client.

As a matter of fact, the use of an external Identiy Provider often blinds developers to the fact that they are still responsible for the User Session, sometimes causing significant security problems to be introduced.

One exception is Azure AD B2B Authentication, which in certain setups will actually handle the user session for you as well.

It is *possible* to build session management on top of Identity Server; basically you have a session store in IdSrv and you use custom middleware to call back to IdSrv on each request to validate the session and obtain the claims for the user. This works, but it is a fair bit of work and probably only right for certain specific scenarios. 


## Design goals
The User Session Manager will be designed to be called *after* the user has been authenticated, whether locally or by some external Identity Provider. It will establish a User Session, which is stored in a server-side session store, and issue a cookie or a token (other options can be introduced). It will also include the appropriate logic to validate the cookie/token on each subsequent request and populate the Claims Principal for the request.
- Fast
- Support for "Remember me"
- Support for both ASP.Net Core and older ASP.Net, the latter probably through OWIN.

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

