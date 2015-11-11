---
layout: post
title: Browser tab tracking
---

Recently I was trying to solve a problem where it would have been quite handy to show if users were actively accessing the site from multiple browser tabs at the same time. So after a bit of thinking and hacking I came up with the following solution.

I now have a small Javascript snippet that given a unique id for a browser tab, inserts it as a hidden field into all FORMs and adds it to the data POSTed with all AJAX requests. This means that I’m not identifying where GETs come from, but that’s fine for my situation as GETs don’t change data. On the server side I’m then extracting this field and including it as a tag in my log against each request along with user and session identifiers. I can then filter my logs by the session and then see if there are interleaved requests with more than one browser tab ID.

That’s all pretty straight forward, obviously the tricky part is identifying the browser tab. I’m doing this by using the [sessionStorage API](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage) where `“Opening a page in a new tab or window will cause a new session to be initiated”`, or in fact simply changing domains in a tab appears to reset the session too.

So the flow is like this: once the page loads the Javascript snippet checks the session storage for a previously stored browser tab identifier and uses that, if it’s not there it stores a newly generated one which is then used. I don’t really trust Javascript GUIDs to be unique, so I’m generating them server side and including them in a META tag.

{% highlight javascript %}
$(function() {

  if (sessionStorage) {
    var btsg = sessionStorage.getItem('btsg') ||
                     $('meta[name=btsg]').attr("content");
    if (btsg) {
      sessionStorage.setItem('btsg', btsg);
      $.ajaxSetup({
        data: { 'btsg': btsg }
      });
      $('form').append('<input type="hidden" name="btsg" value="' + btsg + '" />');
    }
  }

});
{% endhighlight %}

Here `btsg` stands for Browser Tab Session GUID.

As mentioned above, in a Rails layout I have this for if a new ID is needed:

{% highlight erb %}
<meta name=“btsg" content="<%= Guid.new.to_s %>" />
{% endhighlight %}

Rails tagged logging can easily be used to get it appearing in the logs, just stick the following in `application.rb`:

{% highlight ruby %}
config.log_tags = [
  lambda {|req| req.env['rack.request.form_hash'].try :values_at, 'btsg' }
]
{% endhighlight %}
