# nicklesndimes (WEB 200pts) 

We are given a link to a website. Things we can do on it are registration, logging in, resetting a password (using a revovery token sent to an email) and listing other users. The task is to log in as admin. 

One can notice that the site is based on the same framework as the CTF website itself - Mellivora. It can be seen visually or because there are static files mellivora.css and mellivora.js. Real 9447 CTF site even has a label:

The label is a hyperlink to a framework source code repo, which is availabe publicly: https://github.com/Nakiami/mellivora. So, users on the website are shown as CTF teams (with a zero score that never increases). 

The very first strange thing we noticed was the unused function in mellivora.js:
```JavaScript
function whitelist_ip(i, t) {
    $.post("admin/actions/new_ip_whitelist", {
        action: "new",
        user_id: i,
        ip: t
    })
}
```
Unfortunately, adding our ip to a white list with any user id did not make any change - the server just returned OK for any valid user id (positive integer). 

The next idea was that registration could have a flaw allowing to register a user with admin rights, but that didn’t work too.

We also noticed that the challenge website disclosed admin’s email in a JSON-formatted user list, unlike real 9447 CTF site. Password recovery emails also arrive from this email.

We then decided to install the framework to test it both from inside and outside. We started thinking that password recovery was vulnerable and that was a right way to go. A password recovery token looked differently on our test stand with fresh Mellivora installation and on the real CTF web site. By default, function to generate a token looked like this:
```php
$auth_key = hash('sha256', generate_random_string(128));
```
This gives a random hex string of length 64 (32 bytes). The challenge site, on the other hand, sent a token of length 32 (16 bytes) which was actually an MD5 hash of a username! 
We were able to reset admin’s password using a token MD5(‘admin’) and then log in as admin providing his email. The site then showed an error telling that our ip address was not in the whitelist, so we used a function found earlier to add our ip address to whitelist, log in again and finally get a flag.  
