#EZProxy Security Issues

There are a couple of EZProxy security issues that are not addressed in the [Securing Your EZProxy Server](https://www.oclc.org/support/services/ezproxy/documentation/example/securing.en.html) OCLC documentation page. This page attempts to summarize these issues, and suggests possbile steps for remediation. The last section of this page contains links to related threads on the EZProxy message board.  

##Clickjacking

EZProxy will prompt for user authentication credentials from within an iframe. There is no way to prevent a malicious site from mocking up a version of your institution's EZProxy login page, but the fact that it is so easy to accidentally misconfigure systems to frame EZProxy login by default is dangerous. Even in the absence of current malicious activity, users may become accustomed to submitting their credentials in a way that is prone to eventual exploitation. 

As message board link #1 points out, to address this issue, EZProxy could be adjusted to allow configuration of an `X-Frame-Options: DENY` http header on the login page
. In the meantime, the framebusting javascript approach described [here](https://www.owasp.org/index.php/Clickjacking_Defense_Cheat_Sheet#Best-for-now_Legacy_Browser_Frame_Breaking_Script) should be effective in preventing the majority of accidental misconfiguration. 

At the University of Pennsylvania, we have employed a [modified version of the framebusting approach](https://github.com/upenn-libraries/ezproxy-framebust) to accommodate framed proxied content, while preventing framed EZProxy login.

##Session hijacking

EZProxy is inherently vulnerable to session hijacking. Even if steps are taken to protect primary user authentication credentials (e.g., enabling `Option ForceHTTPSLogin`), subsequent session authentication takes place via the EZProxy session cookie, which is sent *in the clear* when accessing non-https proxied sites. 

There is an undocumented and unsupported `Option ForceSSL` which causes all client-proxy connections to be https-encrypted. In practice this option apparently raises cross-domain and/or mixed/insecure content issues that render many resources unusable. (See message board discussions #5 and #6). Additionally, it might be undesirable to use one's institutional SSL certificate to vouch for proxied content that is in fact insecure. 

So with `Option ForceSSL` out of the picture, EZProxy currently requires session tokens to be sent in the clear. For any user on an insecure network (e.g., many public wireless networks), session tokens are therefore vulnerable, and could easily be used to request resources on behalf of the user for the life of the session. 

###Session hijacking mitigation

####Absolute `MaxLifetime`

Add an *absolute* equivalent to the `MaxLifetime` config directive. The existing `MaxLifetime` config directive is relative to the last access time; so, in the absence of manual intervention, the absoulte maximum session lifetime is effectively infinite. 

####Https HMAC preflight validation of insecure requests

In this approach, a sucessful login request would set ezproxy session cookie (as usual), but would additionally set a secure cookie (to be sent only via https to the login port). Insecure http requests would require an extra validation parameter (e.g., an HMAC over the original request URI). If the validation parameter is present and valid, the request is served out normally. If present and invalid, send 4xx response. 

If validation param is not present, the client would be redirected to an https validation path (e.g., 302 Location: https://ezproxy.library.edu/validate?request=[original-request-URI]). Validation would proceed in the presence of a valid secure session token (cookie), the server subsequently responding 302 Location: http://\[original-request-URI\]\[validation-param\]

Standard crypto algorithm caveats would apply (timestamps, nonces, initialization vectors, etc. to avoid replay/chosen plaintext, etc.), but something along the lines of this general approach should allow the prevention of session hijacking in EZProxy. A network middleware implementation could be implemented by proxying access to EZProxy and leveraging the `AcceptX-Forwarded-For` config option, but a native EZProxy implementation would be more straigtforward and performant. 

#####Links to the EZProxy Message Board

1. [EZproxy and clickjacking](http://ls.suny.edu/read/messages?id=3290186) 2015-03-04 (framed login and clickjacking)
2. [Ezproxy firesheep session hijacking?](http://ls.suny.edu/read/messages?id=1470704) 2010-11-02 (http and session hijacking)
3. [Changing http to https in all stanzas](http://ls.suny.edu/read/messages?id=2892356) 2012-08-24 (http and session hijacking)
4. [http vs. https - potential security issue](http://ls.suny.edu/read/messages?id=1005455) 2009-06-25 (http and session hijacking)
5. [https only?](http://ls.suny.edu/read/messages?id=269507) 2008-05-28 (A little background on why Option ForceSSL is unsupported)
6. [SSL](http://ls.suny.edu/read/messages?id=78356) 2004-05-07 (Mentions unsupported Option ForceSSL)
