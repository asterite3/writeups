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

Some experimenting with this URL showed us that changing `hash` or `timestamp`, as well as using the same `hash` and `timespamp` for another `id` would lead to website showing us a message `hey boy`, which must be an error message. So we clearly have a signature which is made at client side and is checked at server side. 

It also turned out that the script will refuse to sign an `id` containing anything but digits. Signing valid numbers was not very useful - only numbers from 1 to 5 gave some responses (not a flag), for others empty response was given. So we started thinking that we have to bypass `id` value checking to be able to sign and send arbitrary `id`s to server - and then expoloit some kind of injection, probably SQLi. Signing happens in Module.main, which is not defined on a page, so it must be in `functionn.js`. It is time to look there.

`functionn.js` turns out to be a large, ~18k-lines script. At the beginning, there are some comments, among them these ones:
```javascript
// 2) We could be the application main() thread proxied to worker. (with Emscripten -s PROXY_TO_WORKER=1) (ENVIRONMENT_IS_WORKER == true, ENVIRONMENT_IS_PTHREAD == false)
```
```javascript
// An online HTML version (which may be of a different version of Emscripten)
//    is up at http://kripken.github.io/emscripten-site/docs/api_reference/preamble.js.html
```
Also, reading further we find tons of hardly-readable code with function names like these
```
__ZN10emscripten8internal7InvokerINSt3__112basic_stringIcNS2_11char_traitsIcEENS2_9allocatorIcEEEEJS8_EE6invokeEPFS8_S8_EPNS0_11BindingTypeIS8_EUt_E
__ZN10emscripten8internal11BindingTypeINSt3__112basic_stringIcNS2_11char_traitsIcEENS2_9allocatorIcEEEEE10toWireTypeERKS8_
```
Ok,so this surely looks like a C++ code comliled to JS with [Emscripten](https://github.com/kripken/emscripten). These names, after removing one underscore at the beginning, can be fed to `c++filt`, giving:
```c++
emscripten::internal::Invoker<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::invoke(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > (*)(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >), emscripten::internal::BindingType<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::{unnamed type#1}*)
emscripten::internal::BindingType<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::toWireType(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)
```
So, if our theory about signature forging is right, we would have to reverse-engineer this code and either find a place where validation is performed to patch it and have the script sign `id`s for us, or understand the signature algorithm and make signatures ourselves.
