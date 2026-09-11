---
title: Laboration 5 - CSRF
---

[Lab description](https://seedsecuritylabs.org/Labs_16.04/PDF/Web_CSRF_Elgg.pdf)

## Environment & tools
The provided SEED lab environment was used, which is a preconfigured virtual machine image running a 32-bit version of Ubuntu 16.04. It is being run in VirtualBox on a 64-bit Linux host.

The vulnerable web application that will be used to demonstrate CSRF attacks is a locally hosted instance of the Elgg webapp, which has been modified to remove any countermeasures that it has in place to defend against CSRF attacks.

## Task 1: Observing HTTP Request
To observe a GET request done on the Elgg webapp, the network tab in the Firefox developer tools was used.

### GET
A request to Elgg's index page was done with the network tab open. For simplicity's sake, the request's header contents were distilled to the following:

```bash
GET / HTTP/1.1
Host: www.csrflabelgg.com
[...]
Cookie: Elgg=59f3p21ohuquk6tf3v8k8f9742
[...]
```

The browser sends a lot of other headers along with the request, including but not limited to headers about what encoding, format, languages are accepted, whether the HTTP connection should be kept alive, and whether the browser should cache the page.

The first line of the request specifies the method of the request (GET in this case), the URI (the index page at / in this case) and the version of HTTP (1.1 in this case). Afterwards is a list of key-value headers, and the ones that are of interest would be the Host header which determines what virtual host we are talking with on the web server, and the Cookie header which contains a list of the current cookie data that is stored in the browser.

### POST
A login request was done to Elgg's login form with the network tab opened. For simplicity's sake, the request's header contents were distilled to the following:

```bash
POST /action/login HTTP/1.1
Host: www.csrflabelgg.com
[...]
Content-Type: application/x-www-form-urlencoded
Origin: http://www.csrflabelgg.com
[...]
Referer: http://www.csrflabelgg.com/
Cookie: Elgg=59f3p21ohuquk6tf3v8k8f9742
```

The method on the first line is now `POST`, and the `Content-Type` of the request is `application/x-www-form-urlencoded`. Checking the request payload shows the parameters we filled in, as well as some hidden form elements that got sent along with it:

```
__elgg_token=lBWmyWdIDcnAfZc9VhhOjQ
&__elgg_ts=1745851144
&username=samy
&password=seedsamy
```

Of interest is also the `Origin` header which is sent in this case after a POST form submission has happened, denoting the origin of the domain that the form was on, as well as the `Referer` header which serves a similar purpose.

## Task 2: CSRF Attack using GET Request
As the endpoint to add a friend in Elgg is done as a GET request, and the instance of Elgg we are performing the laboration on has all countermeasures against CSRF disabled on it, we can simply trigger it by crafting an URL and putting it inside an `<img>` tag.

Boby's user ID is 43, so the following tag would make a request to add him as a friend.

```html
<img src="http://www.csrflabelgg.com/action/friends/add?friend=43">
```

When the request is sent by the browser, it sends the cookies that are saved for the `www.csrflabelgg.com` domain, which includes Alice's cookie. So even though Boby can't steal the cookie from another domain from `www.csrflabattacker.com` (theoretically speaking, according to the security policy that cookies generally have), he can make a request to another domain that makes the browser send the cookies associated for that domain.

Normally Elgg has some additional parameters acting as a token, which are assumed to be unique to a request, expire after some time and cannot be realistically guessed, but as this has been disabled there is no such check. We have performed a cross-site request forgery attack.

## Task 3: CSRF Attack using POST Request
Just like how we can create a CSRF attack for GET requests, we can create a CSRF attack for POST requests by crafting a form and automatically submitting it to a specific URL using JavaScript without any user interaction needed.

What we are supposed to do in the task is to craft a page that edits Alice's profile when Alice opens the page. To find Alice's GUID without being logged in as Alice, Boby can just hover over the Add friend or Send message buttons on Alice's profile and see in the URL hover that shows up in the corner that Alice's GUID is 42.

Based on the code that is provided in the lab description, the following code is put in a file on `www.csrflabattacker.com`:

```html
<script type="text/javascript">
var p = document.createElement("form");
p.action = "http://www.csrflabelgg.com/action/profile/edit";
p.method = "post";
document.body.appendChild(p);
window.onload = function () {
	p.innerHTML = "<input type='hidden' name='name' value='Alice'>"
		+ "<input type='hidden' name='briefdescription' value='Boby is my Hero'>"
		+ "<input type='hidden' name='accesslevel[briefdescription]' value='2'>"
		+ "<input type='hidden' name='guid' value='42'>";
	p.submit();
}
</script>
```

Going onto the page when logged in as Alice, it will submit the form on load and end up redirecting to Alice's page on Elgg with the description edited.

As Elgg needs you to specify the GUID in a hidden form parameter when sending the request, it wouldn't really be possible for Boby to create an attack that would work for all users, and since we're on a different domain we would not have any way of retrieving that kind of information like if it were an XSS attack on the same domain. Trying to remove the GUID parameter leads to a redirect loop and Elgg shows the error "You do not have permission to edit this profile."

## Task 4: Implementing a countermeasure for Elgg
Elgg contains a countermeasure for CSRF attacks in the form of secret token parameters used for sensitive request endpoints, `__elgg_token` and `__elgg_ts`. Checking these was originally commented out, and the relevant code is located in `/var/www/CSRF/Elgg/vendor/elgg/elgg/engine/classes/Ellg/ActionsService.php`.

To reenable CSRF protection, the `return true` that stubs the protection is commented out from the `gatekeeper()` function:

```php
public function gatekeeper($action) {
	//return true;
	[...]
}
```

Going onto the page for the POST request CSRF attack causes a redirect loop, and Elgg shows the error "Form is missing __token or __ts fields". The same thing happens to the GET request CSRF attack, and neither is successful anymore.

The Elgg security token is a hash whose string consists of a site secret salt, as well as the session ID and the current salt, together with a second parameter which assumedly matches the timestamp in the security token. This is also referred to as a cryptographic nonce and would be infeasible for an attacker to produce. Tokens which have already been used are already assumed to be invalidated as well as old tokens expiring after some time, preventing potential replay attacks.
