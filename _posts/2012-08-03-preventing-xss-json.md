---
layout: post
title: Preventing XSS attacks when embedding JSON in HTML
---

At Khan Academy, we recently took the time to go through our 200+ [Jinja2][jinja2] templates and turn on autoescape to reduce the likelihood of falling prey to an [XSS attack][xss]. This gave us an excuse to audit all of our pages for injection holes: Here's one hole [Jamie Wong][jamiewong] pointed out that you might run into when using Ruby's `.to_json` or Python's `json.dumps`.

Suppose you're writing a web app and you want to pass down an untrusted string `username` from the server to your client-side JavaScript code. If you create a Rails template that looks like the following, are you safe from XSS attacks? 

    <script>
        Profile.init({
            username: <%=raw username.to_json %>
        });
    </script>

(Here we use `raw` because we don't want HTML entities in the JavaScript code.) Though we're not exactly including unescaped HTML, there's a subtle injection bug here that has to do with how browsers parse `<script>` tags.

The [HTML spec][html4] says:

> Markup and entities must be treated as raw text and passed to the application as is. The first occurrence of the character sequence "</" (end-tag open delimiter) is treated as terminating the end of the element's content.

Where's the security hole? Consider if `username` was set to:

    </script><script>evil()</script>

This will give us the following HTML:

    <script>
        Profile.init({
            username: "</script><script>evil()</script>"
        });
    </script>

Though the first `<script>` tag doesn't contain valid JavaScript, it doesn't matter -- the second script tag will be read and so `evil()` will be executed.

So what's the fix? In addition to the common character escapes `\"`, `\\`, `\b`, `\f`, `\n`, `\r`, `\t`, and `\uXXXX`, the [JSON spec][json] states that `\/` will be interpreted as a literal slash. That is, in a JSON string literal, you can add a backslash before a slash character without otherwise changing the string. To prevent against this hole, you should replace every occurrence of `</` in your JSON with `<\/` so that the `<script>` tag remains open. (The characters `<` and `/` are valid only within a string literal so the replacement can't affect anything else.)

Regardless of which language you use, you'll probably want to make a helper function to encapsulate this logic:

{% highlight ruby %}
# Ruby
def jsonify(obj)
  obj.to_json.gsub('</', '<\/')
end
{% endhighlight %}

{% highlight python %}
# Python
def jsonify(obj):
    return json.dumps(obj).replace('</', '<\\/')
{% endhighlight %}

As long as you always remember to use the `jsonify` wrapper instead of the built-in JSON serialization, you should be safe from this particular attack.


[jinja2]: http://jinja.pocoo.org/
[xss]: https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)
[jamiewong]: http://jamie-wong.com/
[html4]: http://www.w3.org/TR/html4/types.html#type-cdata
[json]: http://www.json.org/
