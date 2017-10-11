---
layout: post
title: Force automatic reload of &quot;Expires&#58; never&quot; resources (css, js)
---

During website development, you have probably already experienced following situation:

1. Make significant changes to website CSS
2. Deploy changed CSS to webserver / app server
3. Verify changed resource by navigating to website an pressing "Reload"
4. Everything looks allright, you pack your bags and head home

And the story continues... You arrive home and you realize, that your **CSS is not refreshed**. What happened
wrong? You start to investigate. Then you realize, that the resource was **cached by your browser** and
doesn't change on a simple page request, because you are using a long resource expiration.

## Handling resource cache expiration

What if you just decrease duration of cache expiration? You will not use never, but browser clients will
redownload your resources every day.

This change shuld be very straight forward. Depending on your technology stack, you just provide
correct configuration. In [nginx](http://wiki.nginx.org/Main), one day resource expiration would look like this:

~~~
location ~* \.(css)$ {
	expires 1d;
}
~~~

As you can see, it is not very flexible. Either you must always conform to a common filename pattern,
or you will always have to enumerate all your resources by hand. What if you want to see changed resources
in that instant without too much hassle?




## Append URL query parameter to your resource

By adding a **changing query parameter** to your resource's URL you solve these 2 issues,
which were present with previous solution:

* Resources expire at fixed interval and browser is force to reload them even if they didn't change
* Cache expiry configuration changes between different technology stacks

Let's put it to the test. Create a website and add a ```<link>``` to your CSS in the ```<head>``` section.
If you look into this website sources, you will see following stylesheet declaration:

~~~ html
<link rel="stylesheet" href="/public/css/custom.css?201405190923">
~~~


Notice the ```201405190923``` parameter in the end. It's the date and time of last build.

Every frontend technology handles resource output differently, but all have one thing in common:

> Stylesheet resource is linked to the website by a URL. Guess what? You should have full control
  over this URL in all frontend stacks.

By defining a variable, which **changes only from build to build**, you have fine grained control
over stylesheet resources expiration, which now expire as you please and you can distribute them with "Never"
expiry policy.
