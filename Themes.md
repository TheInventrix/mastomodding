# Theme Modification a.k.a. Re-Skinning

(If you're looking to build an entirely new theme, this is not the guide for you.)

## Step Zero

Before you get started with themeing: if you haven't done it before, familiarize yourself with your browser's "development tools". This usually includes an element inspector (which lets you look at each HTML element on the page and its associated CSS), a console (for JavaScript output and error messages), and a way to define temporary local modifications to the CSS.

An easy way to access this if you don't like digging through menus (I don't) is to right click on any random part of the page you're looking at and choose the menu item that says something like "Inspect Element". This feature extremely useful for the CSS part.

## Customizing the CSS

### Step One - Create the CSS Files

*All of the files for Mastodon's CSS are in app/javascript/styles/ - please assume this as the working directory for everything regarding CSS.*

You probably already have an idea of what you want your theme to look like, and the first thing you'll probably want to do is **change the colors**. Fortunately, this part is easy to change - although not so easy to test - because all of the colors in the interface are defined as variables by UI usage type, and the base variables are set with !default so your redefinitions will guaranteed override them. The default variables are all in `mastodon/variables.scss`

Now for the first step! You'll need to make a base file for your theme - we'll call the theme "custom" and the base file `custom.scss` for this guide, but you can name it whatever you like. If you plan on making extensive changes, you can organize them in multiple files in a folder - such as `custom/` - as well.

Your `custom.scss` file needs to include two things:

- The new colors for the variables (either directly in the file or as something like `@import 'custom/colors.scss';`)
- `@import 'application';` to pull in all of the default theme CSS. (If you were doing a full theme rewrite, you would probably not do this step and just `@import` each of your individually rewritten files instead.)

Below that, you'll either put in your customized changes, or you'll put in `@import` lines for each of your files containing the changes. For this guide, we'll assume you're putting the changes directly into `custom.scss`.

### Step Two - Testing your CSS changes

Compiling assets every time you want to test a theme change sucks, but trying to set up and run a development instance sucks even more. This is why being familiar with your browser's element inspector is such a good idea, because you can use that to find exactly what style classes are affecting the element you want to restyle and write in your own code changes locally in your browser to see how they look.

**An important note** about this, however. You need to pay attention to how the classes for the styles are actually defined in the style pane. I got tripped up a couple times because I'd set a style for, say, `.autosuggest-textarea__textarea` but the css actually defined the styles for `.compose-form .autosuggest-textarea__textarea` and thus my style was overridden. The Mastodon stylesheets are not even remotely consistent about when and how they do this, so pay attention.

Once you've collected the changes you want to make to your theme, it's time to add them to your `custom.scss` file!

### Step Three - Writing SASS code

Mastodon uses a CSS compiler that works with SASS or SCSS files. These are slightly different from normal CSS files; they have some extra functions available for doing all kinds of style combinations and whatnot. If you don't know how to do those things, there's only one really relevant syntax you'll want, and that's variables.

For example, I have the following code changing some styling for the front page of my instance:

```css
.community-timeline__section-headline,
.public-timeline__section-headline,
.account__section-headline {
  background: darken($ui-bg-color, 4%);
}
```

You can see I used a variable there which isn't one of the pre-existing color variables. You can not only redefine the existing variables from their defaults, but also set whatever variables you want at the beginning of the file and then use it throughout your style tweaks. It also uses a SASS `darken()` function, but I stole that from the original code. If you're creating new color variables or drastically changing the color balance of the interface, I suggest cross-checking with the default theme files for the items you're changing to see if they have a `darken()` or `lighten()` applied, but it's optional.

Really, *all* the SASS stuff is optional. You can take that plain CSS you put together in step two and just throw it straight into your theme file, hex codes and all, without messing around with variables or anything at all, and it'll work fine.

## Changing Interface Labels

In order to rename things in the interface, you need to either create a new locale file and set it as the default - like cybre.space does - or change the existing locale files themselves, like tootplanet.space does. Both of these are entirely viable approaches, although I'm not sure what creating an entirely new locale file entails. I would assume, however, that for a locale named "custom", you'd need to create and fill out the following files:

```
app/javascript/mastodon/locales/custom.json
config/locales/custom.yml
```

If you take the "change existing locale files" approach, you only need to change the ones in `app/javascript/mastodon/locale/` for the languages you're changing.

## Changing Interface Icons

Pretty much every icon button or column icon in the interface is set in the JavaScript files, so if you want to change any of them, you'll have to dig into those. Fortunately, they are quite easy to change, if not so easy to hunt down where to change them...

Lucky for you, I am here to give you the list!

### Column Header icons

For these, look for the `ColumnHeader` section and change `icon='whatever'` to the name of the FontAwesome icon you want to use instead.

```
app/javascript/mastodon/features/home_timeline/index.js
app/javascript/mastodon/features/notifications/index.js
app/javascript/mastodon/features/community_timeline/index.js
app/javascript/mastodon/features/public_timeline/index.js
app/javascript/mastodon/features/standalone/public_timeline/index.js
```

To change the icons in the navigation bar in the Compose column, and in the Getting Started navigation column, look for the previous icon name in the following files and replace it with the new icon name.

```
app/javascript/mastodon/features/getting_started/index.js
app/javascript/mastodon/features/compose/index.js
app/javascript/mastodon/features/ui/components/tabs_bar.js
```

### Status Interaction Icons

To change the icons for status interactions - for example, changing the Reply button to speech bubbles instead of arrows, or changing Boost to a rocket ship - you need to find and change the appropriate icon name in these files.

```
app/javascript/mastodon/components/status_action_bar.js
app/javascript/mastodon/features/status/components/action_bar.js
```

### Column Interaction Icons

To change the icons for column settings, the < next to Back, and any other icons used generally by column headers and settings, find the appropriate icon name in `app/javascript/mastodon/components/column_header.js` and change it to the name of the icon you want.
