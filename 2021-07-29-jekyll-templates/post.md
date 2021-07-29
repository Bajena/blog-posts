
---
title: Rendering a Liquid template with highlight tags in Jekyll
published: true
description:
tags: ruby, jekyll, contentful
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k3r1tl51lch0z9fw4uee.png
---

While developing a blog in [Jekyll](https://jekyllrb.com/) it may happen that you'll have a need for rendering a custom template. I had this situation when trying to build a code snippet block with [Contentful](https://www.contentful.com/) that'd eventually be rendered in a Jekyll blog.

Jekyll's [docs](https://jekyllrb.com/docs/upgrading/3-to-4/#for-plugin-authors) suggest that you should parse the template like this:
```ruby
content = "<pre>{{ contentToDisplay }}</pre>"
template = Liquid::Template.parse(content)
```

Then you can render the template (passing optionally some variables):
```ruby
template.render('contentToDisplay' => "x = 1")
"<pre>x = 1</pre>"
```

It works nicely as long as long as you don't want to use a Jekyll Liquid tag, like the [highlight](https://jekyllrb.com/docs/liquid/tags/#code-snippet-highlighting) tag that I needed.

When you change the `content` to:
```handlebars
{% highlight ruby %}
{{ contentToDisplay }}
{% endhighlight %}
```

then rendering such template will raise a following exception:
```
undefined method `safe' for nil:NilClass
```

It took me some time to figure out what the problem was, but eventually it turned out that the [Jekyll::Tags::HighlightBlock](https://github.com/jekyll/jekyll/blob/master/lib/jekyll/tags/highlight.rb) class expects that there's `site` in template's context.

So, I ended up having three new problems:
1. What should be the type of the `site` variable?
2. Where do I get the variable from?
3. How do I pass it to the `highlight` tag?

After trials & errors I managed to answer all of these questions:
1. The tag expects `site` to be a `Jekyll::Site`
2. You can get the `site` object by calling `Jekyll.sites.first` (I suppose that in 99.9% of cases there's just one site so you shouldn't worry too much)
3. In order to pass the site context when rendering a template the previous call needs to be converted like this:
```ruby
@template.render!({ 'contentToDisplay' => "x = 1" }, { registers: { site:  Jekyll.sites.first } })
```

That's it! Now my template got rendered correctly and the syntax is nicely highlighted:
```html
<figure class="highlight">
  <pre>
    <code class="language-ruby" data-lang="ruby">
      <span class="n">x</span> <span class="o">=</span> <span class="mi">1</span>
    </code>
  </pre>
</figure>"
```

Hope you find this post useful and that it'll save you some time :)
