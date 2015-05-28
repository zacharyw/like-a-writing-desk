---
layout:     post
title:      "Very Custom Form Builders"
subtitle:   "In Ruby on Rails"
date:       2011-04-15 03:43:00
author:     "Zachary Wright"
header-img: "img/post-bg-02.jpg"
archive:    true
---
<span class="text-muted"><small>Since writing this post, I have released two Ruby gems that implement custom form builders for you. [Bootstrap2-form-builder](https://github.com/zacharyw/bootstrap2_form_builder) is for Twitter Bootstrap 2, and [Bootstrap3-form-builder](https://github.com/zacharyw/bootstrap3-form-builder) is for Bootstrap 3.</small></span>

Custom form builders are a bit of Rails alchemy that can drastically reduce the volume of code in your views and minimize duplication. The idea is that across a single application, most of your forms are going to use very similar markup, and you'll also want them to have a consistent look and feel.

Most developers who have worked with Rails know about the baked in form helper methods, and they may very well be some of the first Rails code you ever wrote. As a reminder, they look like this:

~~~ erb
<dl>
  <%=form_for @post do |form| %>
    <dt>
      <%=form.label :title, :class => "required"%>
    </dt>
    <dd>
      <%=form.text_field :title%>
    </dd>
    <dt>
      <%=form.label :content, :class => "required"%>
    </dt>
    <dd>
      <%=form.text_area :content, :size => "90x35"%>
    </dd>
    <%=form.submit "Submit", :class => "button"%>
  <% end %>
</dl>
~~~

The idea here is that `form_for` is smart enough to figure out which model object you're referencing (in this case, `@post` is probably an instance of a Post model, but you could have named the `@post` variable anything and Rails would have figured out what kind of model it is), and generates HTML with markup that contains this knowledge, which will get passed along to your controller when you click that submit button. This information is critical to understand - it not only removes a layer of "magic" from Rails, but also gives you a better grasp on what your custom form builder will be doing when we write it later. For example, the code above would generate HTML that looks like this:

~~~ erb
<dl>
  <form action="/posts" class="new_post" id="new_post" method="post">
    <div style="margin:0;padding:0;display:inline">
      <dt>
        <label for="post_title" class="required">
          Title
        </label>
      </dt>
      <dd>
        <input type="text" size="30" name="post[title]" id="post_title">
      </dd>
      <dt>
        <label for="post_content" class="required">
          Content
        </label>
      </dt>
      <dd>
        <textarea rows="35" name="post[content]" id="post_content" cols="90">
        </textarea>
      </dd>
      <input type="submit" value="Submit" name="commit" id="post_submit" class="button">
    </div>
  </form>
</dl>
~~~

As you can see, the form helper methods we were using created HTML that knows about the model class (post) and about the attributes of that post (title and content). This is called a resource based form, since it wraps a model. Rails supports non-model forms, but we'll get into those another day. The syntax being used on the HTML name attribute, `post[content]` for example, allows Rails to pass all the attributes you enter on your form to the Posts controller as a hash within the `params` hash. In your controller you can do `Post.create(params[:post])`, and all the attributes will be passed along to your model and created. Magic.

Some developers separate their fields with `<p>` tags or list tags, but it doesn't really matter - you can see how writing this same markup each time you need to create a new form could get old fast. Not only will it consume your valuable time that you could be using to write something useful, but it will also consume space within your views, and if you decide to switch from `<p>` tags to `<dt>`/`<dd>` tags, you're going to have your work cut out for you if you have very many forms at all in your application. Rails (as usual) provides a convenient facility to remove all of this duplication: custom form builders.

The Form(ation)
---------------

In order to utilize a custom form builder we are going to subclass the Rails "FormBuilder":http://api.rubyonrails.org/classes/ActionView/Helpers/FormBuilder.html class and either add new helper methods that we need, or override current ones to do something new. Although this is code that's going to help us in our Rails app, it's good to keep in mind that it's really the dynamic power of Ruby that allows us to perform this alchemy. 

The first thing you'll want to do is create a Ruby file to hold your form builder, which we will be calling "standard_form_builder":

<span class="text-muted"><small>app/helpers/standard_form_builder.rb:</small></span> 

~~~ ruby
class StandardFormBuilder < ActionView::Helpers::FormBuilder
end
~~~

Here we have an empty class that sub-classes FormBuilder - our reasoning for doing this being that FormBuilder has already done most of the work for you. It already contains functionality that knows the name of the object you're building the form for, as well as a reference to the view that lets you use all of the handy view methods like `render` and `content_tag`.

The two most important variables provided by FormBuilder, for our purposes, are `@template` and `@object`. `@template` is the current view, and an instance of "ActionView::Base":http://api.rubyonrails.org/classes/ActionView/ActionView/ActionView/Base.html. You use this object to call any methods you would normally use in an erb file or regular helper (`render`, `content_tag`, etc). `@object` is the model object that you initially passed to `form_for`...in our case, `@post`. It also gives you access to `@object_name`, `@options` (options passed to form for), and `@proc` (block passed to `form_for`). `@object_name` actually is important, it gets passed along to the regular form helpers that you can use without `FormBuilder`, but `FormBuilder` takes care of this for you. That last bit may sound confusing, but just know that helper methods on `FormBuilder`, like `radio_button`, really just act like proxies to another method on `FormHelper` with the same name - all it does is pass along the object name:

### FormHelper ###
~~~ ruby
def radio_button(object_name, method, tag_value, options = {})
  InstanceTag.new(object_name, method, self, options.delete(:object)).to_radio_button_tag(tag_value, options)
end
~~~

### FormBuilder ### 

(I'm just going to hand this work off to `FormHelper`, but I'll take care of the object name for you!)

~~~ ruby
def radio_button(method, tag_value, options = {})
  @template.radio_button(@object_name, method, tag_value, objectify_options(options))
end
~~~

Form builder has several methods like this, including `check_box`, `text_area`, `file_field`, etc. It also has one other attribute you can access, `field_helpers`, which is just an array holding the names of those methods.

Phew, now that all that ground work has been laid, it's time to get to the part we've all been waiting for: adding code to our custom form builder!

Lets go back to our `standard_form_builder` and, using our knowledge of `FormBuilder`, its methods, and its attributes, lets override the `text_field` and `text_area` methods to do some of the work for us.  If you recall, I set my form up within a `<dl>`, or a definition list, with the labels being inside of the `<dt>` element and the input fields being inside the `<dd>` element. Since anything worth writing once is worth automating, lets set up our form builder to do this for us:

<span class="text-muted"><small>app/helpers/standard_form_builder.rb:</small></span>

~~~ ruby
class StandardFormBuilder < ActionView::Helpers::FormBuilder
  def text_field(label, *args)
    @template.content_tag("dt", 
      @template.content_tag("label",
                            label.to_s.humanize,
                            :for => "#{@object_name}_#{label}") + 
    @template.content_tag("dd", super(label, *args))
  end

  def text_area(label, *args)
    @template.content_tag("dt", 
      @template.content_tag("label",
                            label.to_s.humanize,
                            :for => "#{@object_name}_#{label}") + 
    @template.content_tag("dd", super(label, *args))
  end
end
~~~

Lets analyze what we've created here. We've subclassed `FormBuilder` and overridden the `text_field` and `text_area methods`. We're leveraging the `@template` variable to call `content_tag`, which just returns a string of HTML for the element we specify (the first parameter to `content_tag`), and containing the second parameter. The `humanize` method we call on `label` takes a string that looks like "tag_list" and turns it into "Tag list". Lastly, we call `super`, which defers to the `text_field` method on `FormBuilder` and it takes care of the rest of the work for the input field.  As a quick example, this is what the `text_field` method above generates when we call `form.text_field(:title)` on our view:

~~~ html
<dt>
  <label for="post_title">Title</label>
</dt>
<dd>
  <input type="text" size="30" name="post[title]" id="post_title">
</dd>
~~~

As you can see, our `form.text_field` method is automagically creating a label and putting our stuff inside `<dt>` and `<dd>` elements, which means we can rewrite our form like this:

~~~ erb
<dl>
<%=form_for @post do |form| %>
  <%=form.text_field :title%>
  <%=form.text_area :content, :size => "90x35"%>
  <%=form.submit "Submit", :class => "button"%>
<% end %>
</dl>
~~~

Not bad at all. Do note that we had to remove our "required" class since there's no easy way (yet) to get that onto the label. We'll get back to that in a minute. 

At this point we've removed a ton of code from our view, and our `text_field` and `text_area` methods are now easily customized in a single location. However, we still have evil duplication lurking about. For one thing, we have to write the `<dl>` tags on every form, and for another, the text_field method is exactly the same as the text_area method...and that will be true for radio_button and check_box if we choose to override those in our builder, which we should. In addition, I still have to add `:class => "button"` to every submit button, if I'm using that class to "style my buttons":https://github.com/zacharyw/css3buttons. That much duplication should be a crime.

Thankfully, we have an easy way to remove it. Remember how I told you that `FormBuilder` has an attribute called `field_helpers` which contains the names of all those helper methods? We can leverage that and the power of Ruby to dynamically create our new helper methods on our custom form builder:

<span class="text-muted"><small>app/helpers/standard_form_builder.rb:</small></span>

~~~ ruby
def self.create_tagged_field(method_name)
  define_method(method_name) do |label, *args|
    @template.content_tag("dt",
      @template.content_tag("label",
                            custom_label,
                            :for => "#{@object_name}_#{label}", :class => label_class) ) +
    @template.content_tag("dd", super(label, *args))
  end

  field_helpers.each do |name|
    create_tagged_field(name)
  end
end
~~~

This should look familiar. We're using the same code to generate the HTML, but now we are iterating over the `field_helpers` array and using Ruby's `define_method` method to dynamically create the helper methods. We pass the label name and the `*args` to the `define_method` block, and the rest is history.

Next, we want to remove the `<dl>` tags we're using to wrap the form, and we want to actually start using our form builder. We'll take care of both of these in one fell swoop. In order to use your form builder, whenever you call form_for, you just pass in the name of your builder in the `:builder` option, like this:

~~~ erb
<%= form_for @post, :builder => StandardFormBuilder do |form|%>
...
~~~

However...you guessed it. We don't want to duplicate this parameter every time we want to make a form, which could be quite often. So lets add a helper method to application_helper.rb to do it for us:

<span class="text-muted"><small>app/helpers/application_helper.rb:</small></span>

~~~ ruby
def standard_form_for(name, *args, &block)
  options = args.extract_options!
  
  content_tag("div",
    content_tag("dl", form_for(name, *(args << options.merge(:builder => StandardFormBuilder)), &block)), :class => "standard_form")
end
~~~

At first glance this looks very complex, but we can walk through it in a couple of steps. First, since `form_for` accepts a variable number of arguments `*args`, which is an array, we use the `extract_options!` method which removes the last element and returns it if it is a Hash, otherwise it returns an empty hash. Next, down in our content_tags, we use `merge` to add in a new option to this hash, our `:builder` that I just told you about, and then append the options back into the `args` array, and pass the `&block` on as well. We wrap up that form_for call in a `<dl>`, and wrap that in a `<div>` for good measure. We also set a class on the div for styling purposes - `content_tag` takes a third parameter which is HTML options to set on the element you're creating.

We're finally ready to use our form builder. Now, when you want to create a form, instead of using `form_for`, use `standard_form_for`:

~~~ erb
<%= standard_form_for @post do |form| %>
~~~

At this point you're free to go on and start using this form builder. However, this blog post isn't titled "Custom Form Builders in Rails," it's titled "Very Custom Form Builders in Rails." And remember how we lost our `:class => "required"` setting in this transition? Lets go on and turn our form builder into a real powerhouse.

A Very Custom Form Builder
--------------------------

If you have used custom form builders before, or if you decided to go ahead and use the most recent example in this post, you have probably ran into a common problem: the custom form builder removed a lot of duplicated HTML, but it also made your HTML much more rigid. You can't set options on the label any more, including the "required" class. Another issue is that the `humanize` method assumes a 'one size fits all' attitude that doesn't always do what you want: it capitalizes the first letter of your label and makes everything else lower case. This is particularly annoying if you have proper nouns or acronyms in your label.

The solution is to make our custom builder smart enough to understand additional options that we pass in. Here's the code for our new `create_tagged_field` method:

~~~ ruby
...
def self.create_tagged_field(method_name)
  define_method(method_name) do |label, *args|
    options = args.extract_options!

    custom_label = options[:label] || label.to_s.humanize
    label_class = options[:label_class]

    @template.content_tag("dt",
      @template.content_tag("label",
                            custom_label,
                            :for => "#{@object_name}_#{label}", :class => label_class) )+
    @template.content_tag("dd", super(label, *(args << options)))
  end
end
...
~~~

We're using the same `extract_options!` method here to remove the options hash from the end of our arguments array. Next we're going to look for a :label option and a :label_class option within that hash, and assign them to variables. If no label is assigned, we'll default it back to the `label.to_s.humanize` value. We'll use these values to customize the label text and the label class. Using this method, you could write a form like this:

~~~ erb
<%=standard_form_for @person do |form|%>
<%=form.text_field :name, :label => "Full Name", :label_class => "required"%>
<%=form.submit :class => "button"%>
<% end %>
~~~

And this would result in a form with a text input and a label that uses "Full Name" as its text, and has the "required" class.

Next lets override the submit method on our form builder so that it automatically includes the "button" class:

~~~ ruby
class StandardFormBuilder < ActionView::Helpers::FormBuilder
  def submit(label, *args)
    options = args.extract_options!
    new_class = options[:class] || "button"
    super(label, *(args << options.merge(:class => new_class)))
  end
...
end
~~~

In this example we once again extract the options we are passed in, and look for one called :class, which is the css class specified on the view. If there isn't one specified, default to "button". In the true spirit of Rails all of your inputs will now have this class by default, unless you configure otherwise.

We're almost done, but there is one small bit of duplication lurking within our code that isn't entirely obvious. Why should we have to specify that an input is required? If you're doing things right you're already using a `validates_presence_of` validator on your class, so technically our application already knows if a piece of input is required or not. Lets leverage that knowledge in our `create_tagged_field` method:

~~~ ruby
def self.create_tagged_field(method_name)
  define_method(method_name) do |label, *args|
    options = args.extract_options!

    custom_label = options[:label] || label.to_s.humanize
    label_class = options[:label_class]

    if @object.class.validators_on(label).collect(&:class).include? ActiveModel::Validations::PresenceValidator
      if label_class.nil?
        label_class = "required"
      else
        label_class = label_class + " required"
      end
    end

    @template.content_tag("dt",
      @template.content_tag("label",
                            custom_label,
                            :for => "#{@object_name}_#{label}", :class => label_class) )+
    @template.content_tag("dd", super(label, *(args << options)))
  end
end
~~~

Here we've taken the `@object`, which holds our model, and collected an array of all the validation classes assigned to that model for this specific attribute, represented by the label. Please note that we are using the original label passed to the form, not the custom label we created. This label needs to correspond to the attribute on the model. If we find the `PresenceValidator`, we give that label the required class. You could use a very similar technique to concatenate a * character onto your label instead.

We've finally made it to the end. That's about all there is to know about custom form builders in Rails. Using what we've outlined in this post you should be able to make your forms as custom as they need to be. Here is the final custom form builder code, in all of its glory:

The Final Form
--------------

<span class="text-muted"><small>app/helpers/standard_form_builder.rb:</small></span>

~~~ ruby
class StandardFormBuilder < ActionView::Helpers::FormBuilder
  def submit(label, *args)
    options = args.extract_options!
    new_class = options[:class] || "button"
    super(label, *(args << options.merge(:class => new_class)))
  end

  def self.create_tagged_field(method_name)
    define_method(method_name) do |label, *args|
      options = args.extract_options!

      custom_label = options[:label] || label.to_s.humanize
      label_class = options[:label_class]

      if @object.class.validators_on(label).collect(&:class).include? ActiveModel::Validations::PresenceValidator
        if label_class.nil?
          label_class = "required"
        else
          label_class = label_class + " required"
        end
      end

      @template.content_tag("dt",
        @template.content_tag("label",
                              custom_label,
                              :for => "#{@object_name}_#{label}", :class => label_class) )+
      @template.content_tag("dd", super(label, *(args << options)))
    end
  end

  field_helpers.each do |name|
    create_tagged_field(name)
  end
end
~~~

And remember to add this method to app/helpers/application_helper.rb:

~~~ ruby
module ApplicationHelper
  def standard_form_for(name, *args, &block)
    options = args.extract_options!

    content_tag("div",
                content_tag("dl", form_for(name, *(args << options.merge(:builder => StandardFormBuilder)), &block)),
                :class => "standard_form")
  end
...
end
~~~

And finally, to use your form builder in a view:

~~~ erb
<%= standard_form_for @post do |form| %>
  <%= form.text_field :title %>
  <%= form.text_area( :content, :size => "90x35") %>
  <%= form.submit "Submit"%>
<% end %>
~~~ erb

This will automatically generate labels for the two input fields and add a "required" class to them, as well as add the "button" class to the submit button. Remember, you can optionally pass in a `:label` or `:label_class` to the `text_field` and `text_area` methods to customize the label.

Disclaimer: I've tested and used this in Rails 3. It probably works in Rails 2 too, but I'm not 100% sure.

### Resources ###

* "onrails":http://onrails.org/2008/06/13/advanced-rails-studio-custom-form-builder
* "Railscasts Episode #211":http://railscasts.com/episodes/211-validations-in-rails-3
* "Agile Web Development With Rails 3rd Edition":http://pragprog.com/titles/rails3/agile-web-development-with-rails