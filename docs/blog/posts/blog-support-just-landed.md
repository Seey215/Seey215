---
date: 2022-09-12
authors: [squidfunk]
description: >
  Our new blog is built with the brand new built-in blog plugin. You can build
  a blog alongside your documentation or standalone
categories:
  - Blog
links:
  - setup/setting-up-a-blog.md
  - plugins/blog.md
---

# Blog support just landed

__Hey there! You're looking at our new blog, built with the brand new
[built-in blog plugin]. With this plugin, you can easily build a blog alongside
your documentation or standalone.__

Proper support for blogging, as requested by many users over the past few years,
was something that was desperately missing from Material for MkDocs' feature set.
While everybody agreed that blogging support was a blind spot, it was not
obvious whether MkDocs could be extended in a way to allow for blogging as we
know it from [Jekyll] and friends. The [built-in blog plugin] proves that it is,
after all, possible to build a blogging engine on top of MkDocs, in order to
create a technical blog alongside your documentation, or as the main thing.

<!-- more -->

_This article explains how to build a standalone blog with Material for MkDocs.
If you want to build a blog alongside your documentation, please refer to
[the plugin's documentation][built-in blog plugin]._


## Quick start

### Creating a standalone blog

You can bootstrap a new project using the `mkdocs` executable:

```
mkdocs new .
```

This will create the following structure:

``` { .sh .no-copy }
.
├─ docs/
│  └─ index.md
└─ mkdocs.yml
```

#### Configuration

In this article, we're going to build a standalone blog, which means that the
blog lives at the root of your project. For this reason, open `mkdocs.yml`,
and replace its contents with:

``` yaml
site_name: My Blog
theme:
  name: material
  features:
    - navigation.sections
plugins:
  - blog:
      blog_dir: . # (1)!
  - search
  - tags
nav:
  - index.md
```

1.  This is the important part – we're hosting the blog at the root of the
    project, and not in a subdirectory. For more information, see the
    [`blog_dir`][blog_dir] configuration option.


#### Blog setup

The blog index page lives in `docs/index.md`. This page was pre-filled by
MkDocs with some content, so we're going to replace it with what we need to
bootstrap the blog:

``` markdown
# Blog
```

That's it.

### Writing your first post

Now that we have set up the [built-in blog plugin], we can start writing our
first post. All blog posts are written with the [exact same Markdown flavor] as
already included with Material for MkDocs. First, create a folder called `posts`
with a file called `hello-world.md`:

.
├─ docs/
│  ├─ posts/
│  │  └─ hello-world.md # (1)!
│  └─ index.md
└─ mkdocs.yml

