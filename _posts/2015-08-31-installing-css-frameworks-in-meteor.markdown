---
layout:     post
title:      "Installing CSS frameworks in Meteor or:"
subtitle:   "How I learned to stop worrying and love the Atmosphere"
date:       2015-08-31 19:20:00
author:     "Zachary Wright"
header-img: "img/post-bg-03.jpg"
archive:    true
---

A few months ago I started playing around with the [Meteor](https://www.meteor.com/) web framework. Meteor is a JavaScript platform that uses JS for both the front end (client) and the back end (server). The parenthesis aren't meant to insult your intelligence - a core aspect of writing Meteor code is specifying, within one JavaScript file, if your app is tiny, what code should run on the server and what should run on the client.

The great part about Meteor is how it uses WebSockets to push updates out to clients instead of relying on AJAX polling, so it's awesome for applications that need multiple users to collaborate on the same screen and see each other's updates quickly. 

The bad part about Meteor is that it's not really obvious how anything works - you just follow the guides and API and it seems to sort things out. I haven't ran into any issues yet that require me to dive into the core of Meteor, but I'm a little terrified of when that time comes. It also requires MongoDB, which for some people is a deal breaker. 

On my first Meteor project I decided that I wanted to go with a different CSS framework than usual, and opt for [Foundation](http://foundation.zurb.com/) over [Bootstrap](http://www.getbootstrap.com). I had a couple of reasons for doing so: It's always nice to learn new things, and I'm a little tired of the "bootstrap look" that every Bootstrap app has. Foundation has its own default look and feel, but it's more minimal and easier to customize.

My first reaction was to go out and get the Foundation SCSS files and copy them over to Meteor. That immediately didn't work, and after some stubbornness I realized it really wasn't worth trying to get to work. For some reason, Foundation by default wants you to install their framework using a command line tool. That seems extremely unnecessary for a CSS framework - it requires Git, NodeJS, and Ruby... just to install a CSS framework. 

Luckily, or so I thought, there is a [Manual Download](http://foundation.zurb.com/docs/sass.html#nocli) section in the docs. This is barely more manual: you get to copy a couple of files yourself, but then you still need Ruby, Node, compass, and bower to actually install the rest of Foundation. I'm not sure why they don't just create a repo with all the SCSS files. I, personally, prefer to manage those manually rather than using package management like Bower. They're just static files that you include on a web page - having this much pomp and circumstance surrounding them just feels like overkill. 

Regardless, even if these files were easily available, it would be difficult and unmaintainable to use them in Meteor. When a Meteor app is loaded up, it uses a set of rules to determine in what order files are loaded in:

<span class="text-muted"><small>From the Meteor docs:</small></span>

1.  HTML template files are always loaded before everything else
2.  Files beginning with main. are loaded last
3.  Files inside any lib/ directory are loaded next
4.  Files with deeper paths are loaded next
5.  Files are then loaded in alphabetical order of the entire path

As you can see, if you wanted to manually install ANY CSS or Javascript framework, you would have to restructure the files in the order they needed to be loaded based on how deep they are, and by alphabetizing their names appropriately. The exception to these rules are Meteor packages: they can define how their contents are loaded. 

This moment was a bit of an epiphany for me, regarding Meteor. If you're going to use Meteor for your project, you need to go all in on Meteor. This means either utilizing an existing Meteor package for what you need, or building your own. I'm a fan of installing my own JS and CSS dependencies, but not so much of a fan that I want to write my own package to do it, especially not when one already exists.

Atmosphere
----------

[Atmosphere](https://atmospherejs.com) is a repository of Meteor packages. Think of it as the [RubyGems](https://rubygems.org/) of the Meteor world, except instead of using Bundler and a gemfile to manage your packages, it's built into the core of Meteor: A simple `meteor add user:package` will install the specified `package` from the specified `user`. Atmosphere should be the first place you go if you need a framework or library. It's the place you upload one to if you're the first to create it.

I would list packages here, but frameworks are updated and changed so frequently you never know which Meteor user will have the best package at any given time. So if you want to use Foundation, Bootstrap, Handsontable, or any other CSS or Javascript library with Meteor, find a package on Atmosphere, add it, and move on with your life. If it doesn't exist yet, think of it as an opportunity to contribute to the community and expand your open source portfolio. 

For reference, here are the two packages I ended up using as of this writing to get SCSS and Foundation installed with Meteor:

1.  [fourseven:scss](https://atmospherejs.com/fourseven/scss)
2.  [juliancwirko:zf5](https://atmospherejs.com/juliancwirko/zf5)

