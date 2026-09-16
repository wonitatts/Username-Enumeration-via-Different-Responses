# Username-Enumeration-via-Different-Responses

Lab: PortSwigger Web Security Academy — Authentication Tools used: Burp Suite (Proxy, Intruder) Vulnerability class: Username Enumeration + Brute-Force (no rate limiting)

TL;DR

The login page gave different responses depending on whether a username existed — a valid username produced a different error (and a different response length) than an invalid one. That let me confirm a real username (aix) out of a wordlist before ever touching the password. With the username known and no brute-force protection in place, I ran a second automated attack against the password field and spotted the successful login by its 302 redirect status. Logged in with the recovered credentials to solve the lab.

Background: the two ideas behind this attack

Username enumeration is when a site behaves differently for valid vs. invalid usernames, letting an attacker figure out which accounts actually exist. This turns one hard problem (guess a username and password together) into two easy sequential ones: first confirm a valid username, then attack only that user's password.

Brute-forcing is trying many candidate values automatically until one works. It only succeeds when the app fails to limit attempts (no rate limiting, lockout, or CAPTCHA).

The tool for both phases is Burp Intruder — it takes one captured request and fires it repeatedly, substituting a value from a wordlist each time. (It's the automated version of Repeater, which resends one request by hand.)

Steps to reproduce
Phase 1 — Enumerate a valid username
Submitted a junk login to capture the login POST request, and sent it to Intruder.
In the Positions tab, marked only the username value as the injection point (username=§...§), leaving the password fixed. Only the username varies between requests.
In Payloads, loaded the candidate usernames wordlist and started the attack.
Sorted results by Length. Almost every response was the same size (the "invalid username" error). One row stood apart at a different length: aix.
Verified before trusting the length delta — opened the aix response and confirmed the error message had changed from complaining about the username to complaining about the password. That confirms the username is valid.
Phase 2 — Brute-force the password
Reused the same request in Intruder. Moved the § markers off the username (now fixed to username=aix) and onto the password value (password=§...§).
Loaded the candidate passwords wordlist and started the attack.
Watched the Status code column. Failed logins all returned 200 with a large response (~3300–3400 bytes — a full re-rendered login page). One row returned 302 with a tiny response (185 bytes).
That 302 row's payload was the password. Logged in with aix + that password through the browser and reached the account page — lab solved.
Why it works

Enumeration: the app leaked account existence through its responses. A valid username produced a different error message (and therefore a different response length) than an invalid one. That difference is an oracle — a signal the attacker can query to learn something the app shouldn't reveal.

Brute-force: with the username confirmed, nothing limited password guessing — no rate limit, no lockout, no CAPTCHA. An automated wordlist walked straight through.

Reading the signals:

Phase 1's tell was length, because the difference was just the wording of an error message on an otherwise identical page.
Phase 2's tell was status code, because a successful login does something structurally different: instead of re-rendering the login page (200, large), the server issues a redirect (302, tiny — a redirect has almost no body) to send you to your account. Both the status and the length agreed.
How to fix it
Make login responses identical for valid and invalid usernames — same message, same status, same length, and same response timing. Remove the oracle and enumeration dies.
Rate-limit and lock out after repeated failed attempts. Even a perfect wordlist is useless if the app stops accepting guesses.
Enforce a strong password policy so common-password wordlists don't contain the answer. (The account here used a trivially weak password, but the app should have blocked the attack regardless.)
Detection (the defender's angle)
Many failed logins against one username in a short window = brute-force in progress.
One source trying many different usernames (same password, or sequentially) = enumeration or password spraying — ideally caught before the password phase begins.
A burst of failures followed by a success from the same source against one account — e.g. many 200s then a 302 — is the compromise moment. That transition is the single most valuable thing to alert on.

Note the symmetry: the exact signal used to attack here (a 302 in a sea of 200s) is the same signal a defender watches for to detect the attack. Same knowledge, both chairs.

Takeaways
Read the results, don't just skim them. Sorting by Length/Status turns a wall of rows into an obvious outlier.
A failed automated run looks different from a "no result" run. Blank status, zero response time, and an error flag on every row means the requests never reached the server (here: an expired lab session), not that the answer wasn't in the list. Diagnosing that is its own skill.
The skill isn't memorizing lab answers — it's building a library of patterns ("different response = enumeration," "302 = successful login") and getting fast at recognizing which one a situation matches.
