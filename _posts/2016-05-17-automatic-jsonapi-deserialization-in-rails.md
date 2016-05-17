---
title: Automatic JSON API Deserialization in Rails
layout: post
blog: true
---
[JSON API](http://jsonapi.org) is a great standard for designing APIs without
going through the same bikeshedding headakes every API designer usually has to
(representing polymorphic associations, duplicated associated records, etc.).
Since the spec has converged to a stable version, it has gained a strong
support, especially in the rails community.[^1] However, most libraries are
aimed towards _serialization_ (i.e. generating JSON API compliant JSON), not
_deserialization_ (i.e. parsing a JSON API payload into a more ready-to-use
format). Implementers are sort of left to do their own deserialization (for
`POST` requests for instance). We addressed that in ActiveModelSerializers by
providing helper methods, although it never felt like it really belonged there.

I created the [jsonapi gem](http://github.com/beauby/jsonapi) to take care of
the parsing and validating part of handling JSON API payloads,[^2] and moved the
deserialization logic there, before removing it as it did not really belong
there either.

Finally, I took advantage of JSON API defining its own MIME type (the rails
implications of which are described in [Benjamin Fleischer's
blog](http://www.benjaminfleischer.com/journey-of-a-media-type-in-rails-part-1))
to issue a [PR to rails](https://github.com/rails/rails/pull/25050), so
that deserialization happens at the earliest possible time, making the
implementer able to use the params as if they were supplied via plain JSON or
multipart data. That is, upon receiving the following payload:
{% highlight json %}
{
  "data": {
    "type": "posts",
    "attributes" : {
      "title": "Hello",
      "date": "today"
    },
    "relationships": {
      "author": { "data": { "id": "2", "type": "users" } },
      "comments": {
        "data": [
          { "type": "comments", "id": "3" },
          { "type": "comments", "id": "4" }
        ]
      },
      "journal": { "data": null }
    }
  }
}
{% endhighlight %}
the value of `params` within the controller would be:
{% highlight ruby %}
{
  "_type" => "posts",
  "title" => "Hello",
  "date" => "today",
  "author_id" => "2",
  "author_type" => "User",
  "comment_ids" => ["3", "4"],
  "comment_types" => ["Comment", "Comment"],
  "journal_id" => nil
}
{% endhighlight %}

[^1]: [`ActiveModelSerializers`](http://github.com/rails-api/active_model_serializers) and [`JSONAPI::Resources`](http://github.com/cerebris/jsonapi-resources) both have nearly full support.

[^2]: A simple JSON schema is not sufficient here, as the format cannot be fully described as such (for instance the full linkage property).
