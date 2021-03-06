---
layout:     post
title:      "Create an RSS Feed with Feedburner"
subtitle:   "In Ruby on Rails"
date:       2011-04-17 15:56:00
author:     "Zachary Wright"
header-img: "img/burner.jpg"
archive:    true
---
[Feedburner](http://feedburner.google.com) is a free Google service that provides you with tools to analyze traffic among your RSS feeds, and to embed advertisements in those feeds. This works by creating (or "burning") an RSS feed in Feedburner by giving it the URL to the RSS feed on your website, and then Feedburner will give you a new URL to provide to your users. This lets Feedburner act as a proxy between your readers and your feed so that it can analyze traffic on your feed, and if you have a Google Adsense account you can easily integrate ads into your feed as well.

Publish an RSS Feed in Rails
----------------------------

The first thing we need to do in our Rails app is publish a feed. All this really means is that we need to provide an RSS version of whatever view we want our users to be able to subscribe too, in addition to the regular HTML view. 

Lets say, for the sake of this example, that you own a blog called BloggyJenkins and you want your users to be able to use an RSS reader to subscribe to your posts. In your `posts_controller` you probably have an `index` method that retrieves all or some of your posts, assigns it to a variable called `@posts`, and then defers to the `index.html.erb` or `index.html.haml` view to do the rendering of these posts. All we need to do is add a different view called `index.rss.builder`, and if our `posts_controller` gets a request for an RSS file, it will automatically choose to render this view instead of the HTML one.

[Builder](http://builder.rubyforge.org/) is a lightweight Ruby library pre-packed with Rails for building out XML documents, which is all an RSS feed really is. Here's the code for your new view:

<span class="text-muted"><small>app/views/posts/index.rss.builder:</small></span>

~~~ ruby
xml.instruct! :xml, :version => "1.0" 
xml.rss :version => "2.0" do
  xml.channel do
    xml.title "Bloggy Jenkins"
    xml.description "An enlightening blog about stuff"
    xml.link posts_url
 
    for post in @posts
      xml.item do
        xml.title post.title
        xml.description post.content
        xml.pubDate post.created_at.to_s(:rfc822)
        xml.link post_url(post)
        xml.guid post_url(post)
      end
    end
  end
end
~~~

Most of the details are pretty self explanatory: We use the predefined `xml` object to create an RSS feed, where we set the title, description, and a link to our original content. We then iterate over the `@posts` variable and render a new feed item for each one. Each item includes the title, description, publication date, a link to the post, and a guid. The guid is extremely important here: it is the unique identifier that RSS readers will use to determine if a reader has read a particular post or not. This should never change and should not be the same as any other post. The post URL is a good option for us here since it fits that criteria.

Next we just need to add a link to our RSS feed somewhere on our site, usually on the front page in the main menu. The link will look something like this:

`<%=link_to posts_path(:rss)%>`{: .language-erb}

Redirect through Feedburner
---------------------------

This is all you have to do to provide an RSS feed to your users. In order to use Feedburner, first create an account and then burn a feed using the URL generated by the above method - it will probably be something like http://www.bloggyjenkins.com/posts.rss. Once you have this new URL you can start distributing it to your users. 

However, we now face a problem. Users don't *have* to use that URL. If they are clever they can still find the http://www.bloggyjenkins.com/posts.rss URL and use it instead, bypassing Feedburner. We need to tell our Rails app to redirect all traffic *except* Feedburner from our true RSS URL to the Feedburner one. We do this by going into the `index` method on our posts_controller, and telling it to respond differently based on the user agent:


<span class="text-muted"><small>app/controllers/posts_controller.rb:</small></span>

~~~ ruby
def index
  @posts = Post.all
 
  respond_to do |format|
    format.html
    format.rss do
      unless request.env['HTTP_USER_AGENT'] =~ /feedburner/i
        redirect_to 'http://feeds.feedburner.com/BloggyJenkins'
      end
    end
  end
end
~~~

And we're done. All requests not from Feedburner will be directed to the Feedburner URL, but Feedburner will still be allowed to view our original RSS feed.