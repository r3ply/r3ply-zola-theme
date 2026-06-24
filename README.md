# r3ply-zola-theme
A Zola theme that adds comments to static websites.

## Installation

First, clone the theme into a directory named `r3ply` within your Zola website's `themes` directory, e.g.

```
# from within your project's root directory
git clone https://github.com/r3ply/r3ply-zola-theme.git themes/r3ply
```

Then, add the theme to the top of your zola config file

```toml
theme="r3ply"
```

Finally, copy over the `comments` directory from the theme to your site's content directory.

```
cp -r themes/r3ply/content/comments content
```

(_Note: you will need add a template to the `content/comments/_index.md` file_)

Next see [setting up r3ply](#setting-up-r3ply-for-your-project).

## Setting up r3ply for your project

(_Make sure you've [installed](#installation) the theme. See above._)

First, install the r3ply CLI.

```
# install r3ply cli tool
npm install -g @r3ply/cli
```

Then initialize your project with r3ply.

```
# initialize r3ply at project root
re init
```

Then copy this config to `static/r3ply.config.toml`

```toml
#:schema https://r3ply.com/schemas/v0.0.1/config/site.v0.0.1.json
version = "0.0.1"
enabled = true

[[site]]
domain = "site.local.test"
r3ply = "cli.r3ply.test"
signet = "A2hioZg5Y-YnjzgOgs4bQw"
issued = 2026-06-21
label = "CLI"

[comments.email]
enabled = true
"&comment_{}" = "comment.template.md"

[moderation]
enabled = true
github = [ ]
webhook = [ ]

  [[moderation.local]]
  "file_path_{}" = "content/comments/{{ comment.ts_rcvd }}_{{ comment.id[:8] }}-{{ author.pseudonym[:7] }}.md"
  enabled = true
```

Then set your config as the default.

```
# set config as default
re config set-default static/r3ply.config.toml
```

Finally, add this r3ply comment template to static/comment.template.md.

```toml
+++
title = {{ comment.txt[:120] | trim | json_encode }}
authors = {{ [author.pseudonym] | str }}
date = {{ email.date }}
slug = {{ comment.id[:8] | json_encode }}

[taxonomies]
# groups comments by author
commenters = {{ [author.pseudonym[:7]] | str }}
# groups comments by subject
subjects = {{ [comment.subject.path[1:-1]] | str }}
# groups comments by thread (the thread is the comment + its replies, and their replies, etc...)
# "all" is a special thread of all comments (used mainly because each tax. term generates an RSS feed)
threads = {{ ["all", comment.id[:8]] | str }}

# used to groups comments by replies (i.e. a comments direct children)
# first member is always itself, so that an RSS feed is guaranteed to exist
# second member is optionally a parent comment, i.e. this comment is replying to it
replies = {{ [comment.id[:8], ...([comment.subject.fragment[1:9]] if (comment.subject.fragment and comment.subject.fragment[:8] != "#:~:text") else [])] | str }}

[extra.email]
dkim = {{ email.auth.dkim }}
dmarc = {{ email.auth.dmarc }}
spf = {{ email.auth.spf }}

[extra.comment]
document = {{ comment.subject.path | json_encode }}
root = {{ comment.subject.fragment is undefined or comment.subject.fragment[:8] == "#:~:text" }}
in_reply_to = {{ comment.subject.fragment | json_encode if comment.subject.fragment else false }}
ctx = {{ __tera_context | str | json_encode }}
+++

{{ comment.html }}
```

When you're done your static directory should have this

```
static
├── comment.template.md
└── r3ply.config.toml
```

You should now be able to simulate comments.

## Usage

### Adding comments

You can simulate receiving comments by using the r3ply CLI command, e.g.

```
# Saves the comment as a file under `/content/comments/`
re simulate email --subject /demo/ --filter comment --moderate
```

For more information on using the CLI tool run `re --help` or see [the docs](https://r3ply.com/docs/cli/).

### Rendering Comments

Use the r3ply macro to render comments. You simply pass in a list of comments—as [Zola pages](https://www.getzola.org/documentation/content/page/).

For example:

```jinja
{% import "r3ply.html" as r3ply %}

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  {% block comments %}
  {% block choose_comments %}
    {% set comments = get_section(path="comments/_index.md") | get(key="pages") %}
  {% endblock choose_comments %}
  {% block render_comments %}
    {{ r3ply::render_comments_sorted_by_activity(nodes=comments) }}
  {% endblock render_comments %}
  {% endblock comments %}
</body>
</html>
```
The macro will automatically find which of the comments from the list are root comments, as well as render the thread with the most recent activity first.

### Customizing Comments

To customize the rendering you need to add a template called `comments.html` to your project's template directory, with a macro called `write_thread` that accepts a `parent` and `comments` argument.

For example, the basic `write_thread` macro could be modified to use tailwind classes:

```jinja
{% import "r3ply/util.html" as util %}

{% macro write_thread(parent, comments) %}
{% set children = comments | filter(attribute="taxonomies.replies.1", value=parent.slug) | sort(attribute="extra.latest_activity") | reverse %}
<details open class="w-xl">
  <summary class="border-b">
    <span class="inline-flex gap-2">
      <span class="text-red-400"><span class="text-gray-600">#</span>{{ parent.slug }}</span>
      <span>|</span>
      <time datetime="{{ parent.date }}" class="text-blue-400">
        <span class="text-yellow-500">{{ parent.date | date(format="%Y-%m-%d") }}</span>
        <span>{{ parent.date | date(format="%H:%M") }}</span>
      </time>
    </span>
  </summary>
  <article class="my-2">
  {{ parent.content | safe }}
  </article>
  <div class="m-2">
    <a class="px-2 py-1 rounded-xl border text-purple-400" href="{{ util::mailto(to=config.extra.domain | default(value='CHANGE_ME'), subject='subject', body='body') }}">reply</a>
  </div>
  <div class="ml-4 pt-2">
    {% for child in children %}
      {# Note: see comment in theme template for why this is `comments::` instead of `self::` #}
      {{ comments::write_thread(parent=child, comments=comments) }}
    {% endfor %}
  </div>
</details>
{% endmacro  %}
```

With the result being something like

![Screenshot of what resulting code snippet looks like](./Screenshot%202026-06-24%20at%2018.06.47.png)
