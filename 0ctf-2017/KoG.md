# KoG (WEB 146pts) 

We are given a link to a website. The main page is totally non-interactive, the only text on it is "King of Glory Player List hmmmm". This definitely means we should explore web page's source.

There we find out that one JS file is included using a script tag, `functionn.js`. Also, there are two JS functions on a page, one that parses URL query string, `GetUrlParms();` (not interesting) and another one, `go()`:

```javascript
function go()
{
    args = GetUrlParms();
    if(args["id"]!=undefined)
    {
        var value = args["id"];
        var ar = Module.main(value).split("|");
        if(ar.length==3)
        {
            var s = "api.php?id=" + args["id"] + "&hash=" + ar[0] + "&time=" + ar[1];
            $(document).ready(function(){
              content=$.ajax({url:s, async:false});
              $("#output").html(content.responseText);
            });

        }
        if((ar.length==1)&(ar[0]=='WrongBoy'))
        {
            alert('Hello Hacker~');
        }
    }
}
```
It takes query string argument `id` and passes it's value to a function Module.main, then takes it's return value. If this value is a string consisting of three parts separated by pipes ('|'), it takes first two of them and makes GET request to URL `/api.php?id=<our id>&hash=<part 1>&time=<part 2>` (third part ignored). Otherwise, we are shown a string "Hello Hacker~".

After appending "?id=1" to task URL and visiting `http://202.120.7.213:11181/?id=1` we saw that a page really made an HTTP request to URL `http://202.120.7.213:11181/api.php?id=1&hash=d48919d108739a8e17f13074fb494f0b&time=1490117207`, where `hash` parameter indeed looks like some kind of hash and time is a timestamp.

Some experimenting with this URL showed us that changing `hash` or `timestamp`, as well as using the same `hash` and `timespamp` for another `id` would lead to website showing us a message `hey boy`, which must be an error message. So we clearly have a signature which is make at client side and is checked at server side. 

It also turned out that the script will refuse to sign an `id` containing anything but digits. Signing valid numbers was not very useful - only numbers from 1 to 5 gave some responses (not a flag), for others empty response was given. So we started thinking that we have to bypass `id` value checking to be able to sign and send arbitrary `id`s to server - and then expoloit some kind of injection, probably SQLi. Signing happens in Module.main, which is not defined on a page, so it must be in `functionn.js`. It is time to look there.
