---
title: Sass for Web Designers
author: 但江
avatar: danjiang
location: 成都 
category: design
---

![Sass for Web Designers](/images/sass-for-web-designers.jpg)

## Nesting

With Sass, you can nest CSS rules inside each other instead of repeating selectors in separate declarations. The nesting also reflects the markup structure.

{% highlight css %}
nav {
  ul {
    margin: 0;
    padding: 0;
    list-style: none;
  }
}
{% endhighlight %}

Nesting namespaced properties.

{% highlight css %}
font: {
  size: 54px;
  family: Jubilat, Georgia, serif;
  weight: bold;
}
{% endhighlight %}

Reference the current parent selector using the ampersand **&** as a special placeholder.

{% highlight css %}
li a {
  color: blue;

  &.alert {
    color: red;
  }

  &.success {
    color: green;
  }
}
{% endhighlight %}

## Variables

Variables in Sass are defined like regular CSS rules using the **$**. Using the **darken** or **lighten** color function in Sass, we can generate different shades of color that will always be based on the brand palette.

{% highlight css %}
$font-stack: Helvetica, sans-serif;
$primary-color: #333;

body {
  font: 100% $font-stack;
  color: $primary-color;
}
{% endhighlight %}

Using variables to define breakpoints.

{% highlight css %}
$mobile-first: "screen and (min-width: 300px)";
@media #{$mobile-first} {}
{% endhighlight %}

## Mixins

Use mixins to define a group of styles just once and refer to it anytime those styles are needed.

{% highlight css %}
@mixin border-radius($radius) {
  -webkit-border-radius: $radius;
     -moz-border-radius: $radius;
      -ms-border-radius: $radius;
          border-radius: $radius;
}

.box { @include border-radius(10px); }
{% endhighlight %}

Mixin with arguments.

{% highlight css %}
@mixin title-style($color, $background) {
  color: $color;
  background: $background;
}

section.main h2 {
  @include title-style(#c63, #eee);
}
{% endhighlight %}

Retinizing hidpi background images.

{% highlight css %}
@mixin retinize($file, $type, $width, $height) {
  background-image: url('../img/' + $file + '.' + $type);

  @media (-webkit-min-device-pixel-ratio: 1.5),
         (min--moz-device-pixel-ratio: 1.5),
         (-o-min-device-pixel-ratio: 3/2),
         (min-device-pixel-ratio: 1.5),
         (min-resolution: 1.5dppx) {
    & {
      background-image: url('../img/' + $file + '-2x.' + $type);
      -webkit-background-size: $width $height;
         -moz-background-size: $width $height;
              background-size: $width $height;
    }
  }
}
{% endhighlight %}

## Import

Enter the **@import** rule, which Sass extends to allow the importing of multiple SCSS files, merging them into a single CSS file when compiled.

{% highlight css %}
// _reset.scss

html,
body,
ul,
ol {
   margin: 0;
  padding: 0;
}


// base.scss

@import 'reset';

body {
  font: 100% Helvetica, sans-serif;
  background-color: #efefef;
}
{% endhighlight %}

## Extend

**@extend** function to chain together styles that are shared amongst multiple selectors.

{% highlight css %}
.message {
  border: 1px solid #ccc;
  padding: 10px;
  color: #333;
}

.success {
  @extend .message;
  border-color: green;
}

.error {
  @extend .message;
  border-color: red;
}

.warning {
  @extend .message;
  border-color: yellow;
}
{% endhighlight %}

Using placeholder selectors with **@extend**. You might create a class that’s not used on its own, prefixing the class name with a **%** instead of a **.**.

You can nest media queries inside the original declaration, and they will bubble up into their own separate declarations when the stylesheet is compiled.

## Operators

Doing math in your CSS is very helpful. Sass has a handful of standard math operators like **+**, **-**, **\***, **/**, and **%**.

{% highlight css %}
.container { width: 100%; }

article[role="main"] {
  float: left;
  width: 600px / 960px * 100%;
}

aside[role="complimentary"] {
  float: right;
  width: 300px / 960px * 100%;
}
{% endhighlight %}
