---
layout: post
title:  "Helpers for Testing Active Model Serializers in Rails with RSpec"
date:   2016-03-19 6:42:49 -0500
tags: ["Ruby", "Ruby on Rails", "Testing"]
---

Lately I've been building an API with Rails using Active Model Serializers. As I was writing my controller tests I realized that I was going to end up repetitively testing my serializers' behavior unless I found a way to test the serializers in isolation. Here's how I did it:

- Create `spec/serializers` directory.
- Create a SerializerSpecHelper in your `spec/support` directory, or wherever you put your spec helpers.

  {% highlight ruby %}
  module SerializerSpecHelper
    def serialize(obj, opts={})
      serializer_class = opts.delete(:serializer_class) || "#{obj.class.name}Serializer".constantize
      serializer = serializer_class.send(:new, obj)
      adapter = ActiveModel::Serializer::Adapter.create(serializer, opts)
      adapter.to_json
    end
  end
  {% endhighlight %}

- Automatically include this helper in all serializers specs by adding this to your `spec/rails_helper.rb`:

  {% highlight ruby %}

  RSpec.configure do |config|
    # lots of other config code is here
    config.include SerializerSpecHelper, type: :serializer
  end
  {% endhighlight %}

- Your serializer specs can then look like this:

  {% highlight ruby %}
  require 'rails_helper'

  RSpec.describe CoffeeSerializer, :type => :serializer do

    describe "attributes" do
      it "should include size as an attribute" do
        coffee = FactoryGirl.build(:coffee, :grande)
        serialized = serialize(coffee)
        expect(serialized["data"]["attributes"].keys).to eq ["size"]
        expect(parsed["data"]["attributes"]["size"]).to eq coffee.size
      end
    end
  end
  {% endhighlight %}

I use the serializer specs to test that my serializers have the right attributes and relationships defined, and particularly to test behavior that's only defined in my serializers. An example might be that my serializer returns a status attribute that masks the true value of status (due to that being sensitive data I don't want to expose to the user) and instead just displays "successful" or "failed", I'd write a test in the serializer spec that "status" returns only successful or failed, to make sure no one changes that behavior unexpectedly.

I'm serializing according to the JSONAPI which has a little more complex structure, so I actually also have some classes for parsing the response into an object from which I can easily read attributes and relationships, match included resources to their resource identifier in a relationship, and so on. Once I polish those up I'll hopefully include them in another blog post!

Kudos to [Attila Györffy](http://eclips3.net/2015/01/24/testing-active-model-serializer-with-rspec/) for his excellent post on testing Active Model Serializers, from which I took the general method of how to serialize an object with Active Model Serializers outside of a controller.
