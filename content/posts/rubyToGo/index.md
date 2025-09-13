+++
title = "From Ruby to GO"
date = "2025-09-04T18:20:04+02:00"
tags = ["ruby", "go", "hugo", "github", "jekyll"]
categories = ["Tooling"]
showTableOfContents = true
showHeadingAnchors = true
heroStyle = "background"
showHero = true
+++

Hi,

I've started to dislike Python and also Ruby. The dependencies and the lack of compatibility had caused me trouble in the past.
Because of that, I moved my coding from Python to [Go](https://go.dev) and started to enjoy it.
Combined with [Wails](https://wails.io), it has supercharged my projects.

However, my Github Pages are still running Jekyll (Ruby), and I always wanted to change that – but I never had motivation.
One day, after finishing an easy box on TryHackMe, my jekyll installation suddenly refused to start the local server. The reason? Version dependencies issues...
That was finally the push I needed to make the switch!

## Hugo

I never heard of [Hugo](https://gohugo.io) before, but after I discovered it – and realized that it's built with Go, I was excited to migrate from Jekyll to Hugo.
Sure, the phrase --"blazing fast"-- is used a bit too often these days, but this time it truly deserves it.

After a quick search, I found the [Blowfish Hugo Theme](https://blowfish.page), which looked just awesome.

## The Migration

Since both cases involve posts written in Markdown, the migration was very straightforward.
There was only one challenge: my images change depending on the theme. Luckily, I found this [article by Sander ten Brinke](https://stenbrinke.nl/blog/adding-support-for-dark-and-light-images-to-hugo-figure-shortcode/) where he implements this feature to Hugo.

![](/images/rubyToGo/theme_based.png)

So everything is done and ready now.

## Resolution Problem

When zooming in on images, the resolution is poor and the image appears blurry. This only happens in the Safari browser. For example, in Firefox, this problems does not occur.

Here a comparsion, Safari in the front and Firefox on the back:
{{<figure-dynamic
light-src="/images/rubyToGo/light/resolution_comparsion.png"
dark-src="/images/rubyToGo/dark/resolution_comparsion.png"
>}}

You can't see it on Safari right now though... &#128540;

This issues is on track: [GitHub: Gallery Images are not being optimized and have lower quality than originals #1459](https://github.com/nunocoracao/blowfish/issues/1459)

Now enjoy my new page!
