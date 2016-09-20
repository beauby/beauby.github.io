---
title: Extending JSON API
layout: post
blog: true
---
# 1. What is JSON API?

According to its homepage, [JSON API](http://jsonapi.org) is "a specification 
for building APIs in JSON". More specifically, it is the combination of a
format specification for JSON payloads representing (collections of)
_resources_[^1], and a set of rules to interact with (i.e.
create/read/update/delete) those resources.

A resource in this context is an object with a `type`, and an `id` (the pair
identifying uniquely the corresponding resource), and optionally some _fields_,
which could be either _attributes_ (arbitrary objects), or _relationships_
(towards zero, one, or several other resources).

Resources could be, for instance, `Posts`, `Users`, `Comments`[^2].

## 1.1. Your resources form a graph

A graph is a simple mathematical structure that consists of a set of _vertices_,
some of which are pairwise related by _edges_ (so, formally, it is a set `V`
along with a subset `E` of `VxV`).

![A directed graph]({{ site.url }}/assets/images/dag.svg)

In this context, your resources form the set `V` of (labelled) vertices of a
graph whose (directed, labelled) edges are the relationships between resources.

More specifically, a vertex is labelled "primarily" by the `type` and `id` of
the resource, and "secondarily" by its attributes.

Moreover, this graph has the property that the `types` of resources form a
partition of the vertices, and that the pair `(type, id)` of a resource
identifies it uniquely.

## 1.2. JSON API defines a representation for subgraphs of resources

A subgraph of a graph is a subset of the vertices of the graph, along with
some (not necessarily all) of the edges attached to those vertices[^3].

As outlined in the introduction, JSON API defines a format for representing
collections of resources. More specifically, it defines a format for
representing a subgraph of your resource graph[^4] (i.e. some resources, and
some (directly or indirectly) related resources).

The way this subgraph is represented is via its adjacency lists: each resource
of the subgraph appears exactly once, along with its edges (i.e. its
relationships towards other resources in the subgraph).

This representation is nice because it avoids headaches when representing
cycles (for instance when user `A` follows user `B` who in turn follows user `A`),
and avoids redundant information.

## 1.3. JSON API defines rules to add/modify/delete vertices/edges

The spec also defines rules for interacting with endpoints representing resources.
The defined operations are (roughly):

1. `POST /users`: create a new vertex in the graph, labelled by the type `users`,
with edges towards already existing vertices;
2. `PATCH /users/5`: modify some secondary labels (i.e. resource attributes) of
a given vertex, along with replacing some edge sets incident to that vertex
(for instance replacing the set of "posts" of that `user` with a new set of (existing)
"posts");
3. `DELETE /users/5`: delete a given vertex, along with all its incident edges;
4. `PATCH /users/5/relationships/posts`: replace the set of edges labelled "posts" with the
given set of edges;
5. `POST /users/5/relationships/posts`: add one edge to the set of edges labelled "posts"
(incident to that `user`);
6. `DELETE /users/5/relationships/posts`: delete the specified edges (that are incident to
the given user).

# 2. Limitations of JSON API

## 2.1. JSON API does not define rules for modifying multiple edge sets

A first limitation that appears is the impossibility to modify multiple edge sets
(i.e. multiple to-many relationships) of a single vertex in one operation,
except if fully replacing those relationships.

To solve this limitation, the following part of the spec:
{% highlight md %}
Updating a Resource’s Relationships:

If a relationship is provided in the relationships member of a resource object
in a PATCH request, its value MUST be a relationship object with a data member.

The relationship’s value will be replaced with the value specified in this member.
{% endhighlight %}

could be modified into:

{% highlight md %}
Updating a Resource’s Relationships:

If a relationship is provided in the relationships member of a resource object
in a PATCH request, its value MUST be a relationship object with **either** a
data member **or one or both of an added_data member, and a removed_data member**.

**In case a data member is specified, the relationship’s value will be replaced**
**with the value specified in this member.**

**In case an added_data member is specified, the values specified in this member**
**will be added to the relationships’s value. (Note: This should not raise an**
**error if part of the edges already exist).**

**In case a removed_data member is specified, the values specified in this member**
**will be removed from the relationship’s value. (Note: This should not raise an**
**error if part of the edges do not exist).**
{% endhighlight %}

As an example, the following request would be valid:

{% highlight json %}
// PATCH http://api.example.com/posts/5
{
  "data": {
    "type": "posts",
    "id": "5",
    "attributes": {
      "title": "Cool updated title",
    },
    "relationships": {
      "tags": {
        "added_data": [
          { "type": "tags", "id": "4" }
        ],
        "removed_data": [
          { "type": "tags", "id": "2" }
        ]
      },
      "comments": {
        "removed_data": [
          { "type": "posts", "id": "5" }
        ]
      }
    }
  }
}
{% endhighlight %}

whereas it would have taken either one non-practical request by full replacements,
or four requests (one for updating the title, one for adding a tag, one for
removing a tag, and one for removing a comment).

## 2.2. JSON API does not define rules to add subgraphs

An other limitation that appears, is the impossibility to add a whole subgraph
to the resources graph. As described above, vertices can only be added one by one,
and newly added vertices can only have edges pointing towards existing vertices.

Three cases could be considered here:

### 2.2.a) Adding a collection of independent vertices

The spec could be extended to support creation of multiple (unrelated) resources
in one request, in a fairly straightforward way, by allowing the following:

{% highlight json %}
// POST http://api.example.com/posts
{
  "data": [
    {
      "type": "posts",
      "attributes": {
        "title": "Cool title 1",
      },
      "relationships": {
        "tags": {
          "data": [
            { "type": "tags", "id": "4" }
          ],
        }
      }
    },
    {
      "type": "posts",
      "attributes": {
        "title": "Cool title 2",
      },
      "relationships": {
        "tags": {
          "data": [
            { "type": "tags", "id": "1" }
          ],
        }
      }
    }
  ]
}
{% endhighlight %}

### 2.2.b) Adding one (or more) independent trees

The spec could also easily be extended to support creation of (one or more, as
in the previous case) trees (i.e. creating one resource, related to other new
resources (possibly related to other new resources, etc.) - the key being that
none of the new resources should be related to any other resource except their
parent). One possible syntax for this is to simply embed the new resources in
their parent's definition. This is possible because the graph is actually a
tree, so it has no cycles.

As an example:

{% highlight json %}
// POST http://api.example.com/posts
{
  "data": {
    "type": "posts",
    "attributes": {
      "title": "Cool title 1",
    },
    "relationships": {
      "comments": {
        "data": [
          {
            "type": "comments",
            "attributes": { "content": "Nice post" },
            "relationships": {
              "tags": {
                "data": [
                  { "type": "tags", "attributes": { "name": "cool-comment" } },
                  { "type": "tags", "attributes": { "name": "first-comment" } }
                ]
              }
            }
          }
        ],
      },
      "tags": {
        "data": [
          { "type": "tags", "attributes": { "name": "cool-post" } },
        ],
      }
    }
  }
}
{% endhighlight %}

which would result in one post, one comment, and three tags being created - two
tags being attached to the comment, itself attached to the post, and one tag
attached to the post directly.

### 2.2.c) Adding an arbitrary subgraph

The spec could be extended to support addition of one or more (almost) arbitrary
subgraphs. The main problems here are

1. multiple references to a newly created resource, and
2. cycles.

The natural way to solve 2. is to represent the graph by its adjacencies, in
order to be consistent with the rest of the spec. The issue is that newly
created resources do not have an `id` yet, so cannot be referenced. The
obvious solution is to give them a temporary `id`, the only purpose of which
would be to identify yet-to-be-created resources inside the document. Such
temporary `id`s could have a special format, like starting with a reserved
symbol. Those newly created secondary resources would then fall under some
`added_data` toplevel member of the document.

Example:

{% highlight json %}
// POST http://api.example.com/posts
{
  "data": {
    "type": "posts",
    "attributes": {
      "title": "Cool title 1",
    },
    "relationships": {
      "comments": {
        "data": [
          { "type": "comments", "id": "&comment-1" }
        ],
      },
      "tags": {
        "data": [
          { "type": "tags", "id": "&tag-1" }
        ],
      }
    }
  },
  "added_data": [
    {
      "type": "comments",
      "id": "&comment-1"
      "attributes": { "content": "Nice post!" },
      "relationships": {
        "tags": {
          "data": [
            { "type": "tags", "id": "&tag-1" }
          ]
        }
      }
    },
    {
      "type": "tags",
      "id": "&tag-1",
      "attributes": { "name": "funny" },
    }
  ]
}
{% endhighlight %}

### 2.2.d) Modifying a subgraph

Finally, it would be possible to combine the solutions outlined in 2.1 and 2.2.c)
in order to provide a way to modify a resource, while adding new secondary
resources, updating some, deleting some, and updating the base resource's to-many
relationships without replacing them. In order to do that, we could introduce, as
in 2.2.c), an `added_data` (along with a `removed_data` and an `updated_data`)
toplevel members, combined with temporary `id`s.

Thoughts?

[^1]: Not exactly the same as REST resources.

[^2]: It is worth noting at this point that a resource is an abstract entity that is not necessarily the direct representation of an object in your database.

[^3]: Note: a subgraph is clearly a graph itself.

[^4]: Not any subgraph, only those that are the union of subgraphs obtained from a collection of resources of same type, by adding all vertices reachable by a prefix of a collection of edge-labelled paths.
