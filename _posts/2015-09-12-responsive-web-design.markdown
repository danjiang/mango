---
title: Responsive Web Design
author: 但江
avatar: danjiang
location: 成都 
category: design
---

![Responsive Web Design](/images/responsive-web-design.jpg)

## A Flexible Grid-Based Layout

### Font Size

* Don`t use no pixel.
* Set body size to 100%, typically the browser’s default type size of 16 pixels.
* Set h1, h2 size to em, target ÷ context = result, such as 24 ÷ 16 = 1.5, the context is container.

### Width

* The context is container, target ÷ context = result.

### Margin

* The context container, target ÷ context = result.

### Padding

* The context is current element, target ÷ context = result.

## Flexible Image and Media

### Image Tag

* max-width: 100% - let browser automatically resize image
* overflow: hidden - cropped to fit inside its container

### Tiled Background Image

* background-position - transition image base on percentage, target ÷ context = result, image dimension not change

### Background Image

* background-size ( CSS3 ) - resize background image

### Retina Image

* image tag with explicit width and height like 50×50, provide a image with size of 100×100
* base on device resolution provide different images, [Easy retina-ready images using SCSS](http://37signals.com/svn/posts/3271-easy-retina-ready-images-using-scss)

## Media Queries

* ViewPort *\<meta name="viewport" content="initial-scale=1.0, width=device-width" /\>* width is window view size, device-width is device size.
* Each query still begins with a media type ( screen ), drawn from the CSS2.1 specification’s list of approved media types.  (Like all, screen, print ).
* Immediately after comes the query itself, wrapped in parentheses: ( min-width: 1024px ). And our query can, in turn, be split into two components: the name of a feature ( min-width ) and a corresponding value ( 1024px ).
* Use max-width property for large screen to force a solid width, [Experimenting with responsive design in Iterations](http://37signals.com/svn/posts/2661-experimenting-with-responsive-design-in-iterations).
* IE8 and below is not support.
* [Media Queries for Standard Devices](http://css-tricks.com/snippets/css/media-queries-for-standard-devices/).

## Workflow

* Photoshop a page mockup, typically a desktop-centric design.
* Turn mockup into html, change browser window size to test.
* Test on devices and laptops to gather issues.
* Change page mockup, treat it as a blueprint to design other pages.
