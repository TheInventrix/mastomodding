***Work In Progress***

## Customizing Post Formatting

Before we get started, I want to be up-front about something very important: I am not going to be teaching regular expressions in this post. If you want to implement things like markdown in Mastodon, you will need to either learn or look up enough about regular expressions to identify the markdown characters on your own.

This guide will explain different approaches for formatting the contents of posts and how they work, but not necessarily the specific way to implement the formatting you specifically have in mind.

### Front-end Formatting

It's possible to apply formatting within posts with a combination of JavaScript and CSS, using `app/javascript/mastodon/components/status_content.js` and a custom theme file. The JavaScript file allows you to alter the displayed HTML essentially however you want, without altering the actual contents of a status in the database, or how it federates to other sites.

The object being sent to `status_content.js` or whatever it is has a lot of information in it - if all the information you need is in the status already somewhere, your job is relatively easy. For example, I have a piece of code that adds a custom class to mentions of local accounts. It does this because the mentions are all mention objects, and within the mention object is an "acct" value with the local username or username@domain of the mention. Since a local username doesn't have an @ in the "acct" value, I just checked if there was a @ character and if not, added the custom class to the link.

```
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

### Back-end Formatting

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

### When to use Front-end versus Back-end

This part is solely my opinion, but here's my personal breakdown.

Front-end if:
- You're extending the theme to have more granularity
- You're doing custom, possibly-complex formatting based on pre-existing information within the statuses
- You don't want your formatting to affect the data being sent to other servers.

Back-end if:
- The custom formatting should be affected by account settings
- You want the formatting to be sent to other servers
