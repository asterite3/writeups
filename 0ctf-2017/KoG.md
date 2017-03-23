# KoG (WEB 146pts) 

We are given a link to a website. The main page is totally non-interactive, the only text on it is "King of Glory Player List hmmmm". This definitely means we should explore web page's source.

There we find out that one JS file is included using a script tag, `functionn.js` (in case the task is disabled, it is saved [here](https://raw.githubusercontent.com/asterite3/writeups/master/0ctf-2017/functionn.js)). Also, there are two JS functions on a page, one that parses URL query string, `GetUrlParms();` (not interesting) and another one, `go()`:

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
## A tiny intro to emscripten-compiled code

In short, code translated by Emscripten works like this:
* Functions are expressed by normal JS functions with all variable and argument names are prefixed with dollar sign.
* Class methods are given `$this` as a first argument.
* Also, sometimes functions will return a value not using JS function return value, but via the special argument `$agg$Result`, which also comes before normal arguments. So, class method returning a value this way looks like
```javascript
function __ZNK5HASH19hexdigestEv($agg$result,$this) {
```
Which is really (in C++) named 
```c++
HASH1::hexdigest() const
```
Wait, what was it's name again?? Alright, we'll get back to it soon/
* Memory is modeled with [ArrayBuffer](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer):
```javascript
var buffer;
buffer = new ArrayBuffer(TOTAL_MEMORY);
HEAP8 = new Int8Array(buffer);
HEAP16 = new Int16Array(buffer);
HEAP32 = new Int32Array(buffer);
HEAPU8 = new Uint8Array(buffer);
HEAPU16 = new Uint16Array(buffer);
HEAPU32 = new Uint32Array(buffer);
HEAPF32 = new Float32Array(buffer);
HEAPF64 = new Float64Array(buffer);
```
So different `HEAP`s are views of this memory array and when a program wants to read one byte from memory at address `$79`, it will do
```javascript
$88 = HEAP8[$79>>0]|0;
```
And for reading four-byte `int` from address `$67`:
```javascript
$90 = HEAP32[$67>>2]|0;
```
Note that for heap views storing memory in larger chunks an address has to be right-shifted, e. g. divided by `size_of_chunk / size_of_one_byte`.

* There is also a stack, residing somewhere on that `HEAP`:
```javascript
 sp = STACKTOP;
 STACKTOP = STACKTOP + 96|0;
 $a = sp + 80|0;
 $b = sp + 8|0;
 $c = sp + 4|0;
 $d = sp;
 $x = sp + 16|0;
 $0 = ((($this)) + 76|0);
 $1 = HEAP32[$0>>2]|0;
 HEAP32[$a>>2] = $1;
 $2 = ((($this)) + 80|0);
 $3 = HEAP32[$2>>2]|0;
 HEAP32[$b>>2] = $3;
```
* Libc and standard C++ library also was statically compiled and implemented in JS in the same way.

## Back to the task

If our theory about signature forging is right, we would have to reverse-engineer this code and either find a place where validation is performed to patch it and have the script sign `id`s for us, or understand the signature algorithm and make signatures ourselves. To our luck, most of the function names and some variable names were preserved, so we did not have to reverse-engineer every function to undestand what it does (`_memcpy` or `_strlen` for example).

We used JS debugger in Google Chrome DevTools to trace through the code execution and undestand what's going on. Another trick that helped us was cloning the page and the script and opening it from the local web server so that we could modify the source code and re-run it.

We also used some helper functions to read memory (`HEAP`):
```javascript
r = function (addr, length){ // read memory region from addr to addr+length as askii string
    var res = [], c;
    for (var i = 0;i < length; i++){
        c = HEAPU8[addr + i];
        if (c == 0) {
            c = 0x380
        }
        res.push(String.fromCharCode(c));
    }
    return res.join('')
}
p = function (addr, length){ // read memory region from addr to addr+length as hex string
    var res = [], c, cc;
    for (var i = 0;i < length; i++){
        c = HEAPU8[addr + i];
        cc = c.toString(16);
        if (cc.length < 2) {
            cc = '0' + cc;
        }
        res.push(cc);
    }
    return res.join('')
}
```

After some stepping in debugger we found out that all intersting stuff happens inside `__Z10user_inputNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEE($agg$result,$inStr)` aka `user_input(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)`.
It takes one argument, c++ `string` (apparently, our user input, `id`), does something with it and returns a result via special argument `$agg$result`.

This function was doing several things. First of all, we noticed it took current timestamp and put it into a c++ string:
```javascript
 (_time(($timestamp_sec|0))|0);
 $10 = (__ZNSt3__111char_traitsIcE6lengthEPKc(1524)|0);
 __ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6__initEPKcj($s,1524,$10); // constructor string (const char* s, size_t n);
 ;HEAP32[$te>>2]=0|0;HEAP32[$te+4>>2]=0|0;HEAP32[$te+8>>2]=0|0;
 $11 = HEAP32[$timestamp_sec>>2]|0;
 HEAP32[$vararg_buffer>>2] = $11;
 (_sprintf($t,1533,$vararg_buffer)|0);
 (__ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6assignEPKc($te,$t)|0); // string::assign(const char *)
 ```
The code works a lot with c++ strings, which confused us for some time. We realized we would have to write another helper that prints contents of c++ string given it's address, and, after reading string constructor (`_ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6__initEPKcj`) we were able to do that.
```javascript
rs = function (addr) {
    var addr32 = addr >> 2,
        isInlined = HEAP8[addr] & 1
        len;
    if (!isInlined) {
        len = HEAP32[addr32 + 1];
        return r(HEAP32[addr32 + 2], len);
    } else {
        len = HEAP8[addr] >> 1;
        return r(addr + 1, len);
    }
}
```
If a string is shorter than 11 characters, it will be inlined - the content will be right inside the object, starting from the second byte.

After taking a timestamp and putting it into a string, function `user_input()` performed some sort of validation on the input string.

Then something interesting happens. The code initializes a string `"Start_here_"` and passes it to function `hi1(std::string)`, looking like this:
```javascript

function __Z3hi1NSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEE($agg$result,$str) {
 $agg$result = $agg$result|0;
 $str = $str|0;
 var $hi1 = 0, label = 0, sp = 0;
 sp = STACKTOP;
 STACKTOP = STACKTOP + 112|0;
 $hi1 = sp;
 __ZN5HASH1C2ERKNSt3__112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE($hi1,$str); // HASH1::HASH1(std::string)
 __ZNK5HASH19hexdigestEv($agg$result,$hi1); // HASH1::hexdigest() const
 STACKTOP = sp;return;
}
```
By looking both at the source of HASH1 and at it's return value we can tell it's `md5`. 

