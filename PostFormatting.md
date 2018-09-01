# Customizing Post Formatting

Before we get started, I want to be up-front about something very important: I am not going to be teaching regular expressions in this post. If you want to implement things like markdown in Mastodon, you will need to either learn or look up enough about regular expressions to identify the markdown characters on your own.

This guide will explain different approaches for formatting the contents of posts and how they work, but not necessarily the specific way to implement the formatting you specifically have in mind.

## Front-end Formatting

It's possible to apply formatting within posts with a combination of JavaScript and CSS, using `app/javascript/mastodon/components/status_content.js` and a custom theme file. The JavaScript file allows you to alter the displayed HTML essentially however you want, without altering the actual contents of a status in the database, or how it federates to other sites.

The object being sent to `status_content.js` or whatever it is has a lot of information in it - if all the information you need is in the status already somewhere, your job is relatively easy. For example, I have a piece of code that adds a custom class to mentions of local accounts. It does this because the mentions are all mention objects, and within the mention object is an "acct" value with the local username or username@domain of the mention. Since a local username doesn't have an @ in the "acct" value, I just checked if there was a @ character and if not, added the custom class to the link.

```javascript
...
      if (mention) {
        link.addEventListener('click', this.onMentionClick.bind(this, mention), false);
        link.setAttribute('title', mention.get('acct'));
+       if (String(mention.get('acct')).indexOf('@') < 0) {
+         link.classList.add('local-mention');
+       }
      } else if (link.textContent[0] === '#' || (link.previousSibling && link.previousSibling.textContent && link.previousSibling.textContent[link.previousSibling.textContent.length - 1] === '#')) {
        link.addEventListener('click', this.onHashtagClick.bind(this, link.text), false);
      } else {
...
```

If, on the other hand, you want to be able to access something that's not already written to be in the objects here, you'll have a lot more work on your hands tracking down whether it's accessible at any other stage of the page-generation process and passing it accordingly.


### What's in the file?

The main thing you want to know to start with is that the entire thing is built around processing a single `status` object. This object contains several things, defined throughout the file to variables for use with the HTML generation, but I'm just going to be looking at the ones most relevant for doing formatting.

```javascript
    const links = node.querySelectorAll('a');
...
    const content = { __html: status.get('contentHtml') };
    const spoilerContent = { __html: status.get('spoilerHtml') };
    const directionStyle = { direction: 'ltr' };
    const classNames = classnames('status__content', {
...

```

`links` (defined on line 33 as of v2.4.4) pulls all of the links from the status; the function it's contained in, `_updateStatusLinks ()` processes the links in a number of ways. External links, mentions, and hashtags are all considered links here and are processed by that function; that's where I added the three lines of code I showed above, that add a special class to mentions of local accounts. If you want to do any kind of special processing of mentions or hashtags, that's the function to do it in.

`content` is pretty self-evident. It contains the actual HTML of the status' contents - ALL of it. If you wanted to do formatting based on characters or such present within the contents of the post - this is what you need to be processing. Which means you'll either want to set it to a variable or do the processing within the assignation itself - probably by defining a function, calling it in the assignment, and then wrapping the existing assignation into the new function.

`spoilerContent` is quite simply the contents of the spoiler tag, or Content Warning. If you want to have customized formatting within the spoiler tag, you'll want to be processing this.

`classNames` is the classes that apply to the *entire* status. Chances are slim you'll want to mess with these, unless you want to style an entire status by some criteria, like making everything in it italicized or all the text green.

Changing what's visible outside of the spoiler tag when the tag is collapsed (i.e. the status contents are hidden) is done with the code starting on line 141 (as of v2.4.4). By default, the only thing that's visible outside of the CW is the list of mentions, but you can define other constants in this section to display when the status is hidden, along with any additional special formatting you want on it.

The part that actually controlls what's visible outside of the spoiler tag is this:

```javascript
      if (hidden) {
        mentionsPlaceholder = <div>{mentionLinks}</div>;
      }
```


## Back-end Formatting

Applying formatting in the Ruby back-end is a *little* bit simpler, in the sense that it only really requires you to work with two .rb files, rather than one .js file, one .scss file, and possibly an endless cascade of .js files if you try to do something really complex. But it also is where local statuses get formatted before being federated, so any changes to the formatting here will be sent to other instances as well.

The two files you'll be working with are:

```
app/lib/formatter.rb
app/lib/sanitize_config.rb
```

`app/lib/formatter.rb` is where the meat of it happens. This is the file that actually parses and formats posts for display and federation. It's important to note that local posts are stored in the local database *without* any of this formatting, while remote posts are stored in the local database as they were received - that is, *with* all of the formatting that was applied by the remote instance.

`app/lib/sanitize_config.rb` controls what formatting is allowed via a whitelist. If you are applying any custom classes or other attributes via `formatter.rb` or applying any classes to tags besides `<span>` or `<a>`, then you will need to add those tags, classes and attribute values to the whitelist.

Changing stuff with `formatter.rb` is a little tricky, because you have to work with all the other formatting that's going on and make sure you apply your changes in the right place in the order and all those other Fun Things, so make sure you read all the functions (or method? are they called methods in Ruby?) in the file to see what they're doing.

There are two *big* advantages to doing your formatting here in the back-end - although one of them could be seen as a disadvantage, depending on your end goal.

- The applied HTML federates to other sites, so if you intend to share the formatting feature you write around, those other instances will have support for displaying the customized formatting and vice versa.
- It's in the Ruby back-end, so it has access to all the ActiveModel stuff! If you're not familiar with Ruby on Rails, what this means is: you can read stuff from the database!

### What's in formatter.rb

```ruby
    html = raw_content
    html = "RT @#{prepend_reblog} #{html}" if prepend_reblog
    html = encode_and_link_urls(html, linkable_accounts)
    html = encode_custom_emojis(html, status.emojis) if options[:custom_emojify]
    html = simple_format(html, {}, sanitize: false)
    html = html.delete("\n")
```

This is where the formatting is actually done on local posts, and where you'll add your own formatting function. What you're formatting and how will affect where you want to put your function in this, to some degree. For example, if you want to do something like I did and style a line if it starts with a particular character, you want to make sure to format *after* `simple_format` - because that's where it adds the `<p>` tags you'll most likely want to put your class onto.

If you're trying to make some sort of markdown formatting, you should most likely do it right after `html = raw_content` and before any of the other tags have been applied.

It's pretty unlikely you'll want to mess with any of the preexisting functions defined in this file; they're mostly just basic encoding of HTML entities and creating links. The three that you're most likely to want to mess with are all the way at the bottom of the file and are the ones that generate the HTML for links - including the applied classes.

```ruby
 def link_html(url)
...
 def hashtag_html(tag)
...
 def mention_html(account)
```

The names are pretty self-explanatory, as are the contents. Just remember when changing these, this affects the way statuses from your instance look when sent to other instances, not just how they look on your site.

### What's in sanitize_config.rb

```ruby
      class_list.keep_if do |e|
        next true if e =~ /^(h|p|u|dt|e)-/ # microformats classes
        next true if e =~ /^(mention|hashtag)$/ # semantic classes
        next true if e =~ /^(ellipsis|invisible)$/ # link formatting classes
      end
```

This section (line 13 as of v2.4.4) is where the allowed classes for *any* internal status elements are defined. If you're attaching special classes to elements within the status, you will want to add a line here containing them.

For example, my RP-styling code adds:

`next true if e =~ /^(thought_bubble|speech_bubble|out_of_character)$/ #rp classes`

```ruby
      elements: %w(p br span a),

      attributes: {
        'a'    => %w(href rel class),
        'span' => %w(class),
      },

      add_attributes: {
        'a' => {
          'rel' => 'nofollow noopener',
          'target' => '_blank',
        },
      },
```

This section (line 23 as of v2.4.4) defines the rest of what's allowed; there's more after it, but it's things like protocols and embeds, which are not really relevant to the question of formatting status text.

`elements` defines which tag elements are allowed within the status contents. If you want to use any standard style tags such as bold or italic, add them here.

`attributes` defines which attributes are allowed to be set for which elements. If you want to set a class on a paragraph element, you'll need to add it here, or if you want to put any other attributes into the allowed elements.

`add_attributes` does a simpler version of the earlier class whitelisting. If you just need one or two possible attribute values added to an element, you can define them here in the same way it's already defined for links.

## When to use Front-end versus Back-end

This part is solely my opinion, but here's my personal breakdown.

Front-end if:
- You're extending the theme to have more granularity
- You're doing custom, possibly-complex formatting based on pre-existing information within the statuses
- You don't want your formatting to affect the data being sent to other servers.

Back-end if:
- The custom formatting should be affected by account settings
- You want the formatting to be sent to other servers
