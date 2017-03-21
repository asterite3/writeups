# KoG (WEB 146pts) 

We are given a link to a website. The main page is totally non-interactive, the only text on it is "King of Glory Player List hmmmm". This definitely means we should explore web page's source.

There we find out that one JS file is included using a script tag, `functionn.js`. Also, there are two JS functions on a page, one that parses URL query string, `GetUrlParms()` (not interesting) and another one, `go()`.