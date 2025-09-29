---
layout: post
title:  "Action Mailer and Phlex views"
date:   2025-09-29
categories: rails
---

I think I might have come up with a design to use Phlex views from Rails'
Action Mailer in a way I'm happy with. This turned out to be quite simple to
code in the end, but since ✨simplicity is hard✨, it took me a bit of
reasoning to get there, so I want to share the results!

First of all, the interface. I wanted to be able to define an email message
like this:

{% highlight ruby %}
module Views::Mailers::Users::Welcome
  include Mailable

  def initialize(user:)
    @user = user
  end

  class Html < Phlex::HTML
    def view_template
      p { "Welcome, #{@user.name}!" }
    end
  end

  class Text < Phlex::HTML
    def view_template
      plain "Welcome, #{@user.name}!"
    end
  end
end
{% endhighlight %}

I was inspired by [this](https://camillovisini.com/coding/phlex-for-rails-emails-action-mailer-without-erb){:target="_blank"}
blog post at first, but my final implementation ended up diverging quite
heavily from what described there.

All mail parts are Phlex components namespaced to a single module representing
the whole message. This allows for convenient editing of content when composing
multipart emails.

On top of this, I wanted every method defined in the main module to be shared
and available to the nested component classes, to allow for DRYer code (super
useful e.g. for `initialize`). Lastly, I wanted every component to be rendered
inside a default (but overridable) implicit layout.

The other side of the message interface is the shape of methods available to
calling code (e.g. `ApplicationMailer` subclasses). My idea for that was
something like this:

{% highlight ruby %}
class UsersMailer < ApplicationMailer
  def welcome(user)
    mail(to: user.email_address) do |format|
      message = Views::Mailers::Users::Welcome
      format.text { render message.text(user:) }
      format.html { render message.html(user:) }
    end
  end
end
{% endhighlight %}

No surprises here, just module methods named along the parts content type,
mirroring the ones exposed by the `format` parameter of the block. Behind the
scenes, they act as factories for mail part instances, so what they do is
basically just forward arguments and instantiate/return Phlex components.

Bonus point: the module exposes methods only for actually defined components
(e.g. no `Html` component > no `html` method). I like to be strict.

So, how did everything fit together? Well, all of the above is implemented by a
<30 lines concern that I called `Mailable`:

{% highlight ruby %}
module Mailable
  extend ActiveSupport::Concern

  class_methods do
    private
      def const_added(const_name)
        return unless part = part_by_name(const_name)
        part.include self
        define_singleton_method(const_name.downcase) { |**kw| part.new(**kw) }
        private_constant const_name
      end

      def part_by_name(name)
        return unless name.in? %i[ Html Text ]
        part = const_get(name, false)
        part if part.instance_of?(Class) && part < Phlex::HTML
      end
  end

  private
    def around_template
      part_name = self.class.name.demodulize
      layout = "Components::Layouts::Mailers::#{part_name}".constantize
      render layout.new { super }
    end
end
{% endhighlight %}

It works by tapping into the `const_added` hook and doing some metaprogramming
magic in order to make the including module a little bit smarter.

As of now it supports plain text and html content types, but the general idea
can be extended. The code should be quite easy to follow, but if you have
questions, please [ask](https://bsky.app/profile/sang.io){:target="_blank"}
away.

I'm really happy about how little code went into the concern, and how tidy and
self contained the overall solution feels like (at least to me).

I hope this helps! And if you have any opinions about this approach and want to
discuss them, I'm [here](https://bsky.app/profile/sang.io){:target="_blank"} to
chat 🎉

Until next time!
