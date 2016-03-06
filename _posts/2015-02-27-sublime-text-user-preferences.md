---
layout: post
title: Sublime Text User Preferences
description: Every time I set up a new dev environment, what do I prefer to set my preferences to.
<!-- modified: 2015-02-23 -->
tags: [sublime text]
image:
  feature: sublime-text-user-preferences.jpg
  credit:
  creditlink:
---

Every time I set up a new dev environment, I prefer to set my preferences to comfortable defaults.

{% highlight ruby %}
  # User Preferences
  {
    "always_show_minimap_viewport": true,
    "auto_complete_commit_on_tab": true,
    "codeintel_scan_files_in_project": true,
    "color_scheme": "Packages/User/SublimeLinter/Monokai (SL).tmTheme",
    "ensure_newline_at_eof_on_save": true,
    "find_selected_text": true,
    "font_face": "Ubuntu Mono",
    "font_size": 12,
    "highlight_active_indent_guide": true,
    "highlight_line": true,
    "highlight_modified_tabs": true,
    "hot_exit": false,
    "ignored_packages":
    [
      "Vintage"
    ],
    "tab_size": 2,
    "translate_tabs_to_spaces": true,
    "trim_trailing_white_space_on_save": true,
    "word_wrap": true
  }
{% endhighlight %}

The `font-face` and `color_scheme` settings do vary depending on the OS, but the others give me a familiar, useful environment.

I map a couple of key-bindings to make pasting and reindenting a little easier.

{% highlight ruby %}
  # User Key Bindings
  [
    { "keys": ["super+v"], "command": "paste_and_indent" },
    { "keys": ["super+shift+v"], "command": "paste" },
    { "keys": ["super+shift+r"],  "command": "reindent" }
  ]
{% endhighlight %}

I also use a set of common packages.

{% highlight ruby %}
  # Package Controller Settings
  "installed_packages":
  [
    "All Autocomplete",
    "BeautifyRuby",
    "ERB Snippets",
    "Git",
    "GitGutter",
    "Haml",
    "Markdown Extended",
    "Maybs Quit",
    "Package Control",
    "Sass",
    "SideBarEnhancements",
    "SublimeCodeIntel",
    "SublimeLinter",
    "SublimeLinter-jshint",
    "SublimeLinter-ruby",
    "SyncedSideBar"
  ]
{% endhighlight %}





















