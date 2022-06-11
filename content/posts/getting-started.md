---
title: "Hugo - New Start"
date: 2022-03-24T11:25:21+02:00
draft: false
cover:
    image: "/posts/images/hugo-logo-wide.jpg"
    alt: "Hugo - New Start"
---

I rebuilt my website from scratch using [Hugo](https://gohugo.io) instead of using WordPress. On one side, to save some of the hosting costs. On the other side, I wasn't happy with the limitations imposed on me by WordPress.

Don't get me wrong, if you want to quickly and easily make a website, then WordPress is usually a good choice. But that's not what I'm after. My personal website consists of static pages, will be updated on a once or twice per month at the most, and does not require any fancy features. This means that I don't really need all the extra bells and whistles. With Hugo, I have full control over the website content, and I can optimize the pages to *only* contain what I need.

Let me tell you, it wasn't easy to rebuild the whole blog, but it wasn't too complicated either. I spent a couple of days copying and pasting over the content, adjusting it to markdown syntax, and learning little tricks from [Hugo's online documentation](https://gohugo.io/getting-started/).

I'm very happy with the trip so far. Plus, the performance improvements in site load and render times are very noticeable. And that makes me even happier.

Here's how I transitioned my website from WordPress to Hugo. Disclaimer: this guide is for macOS but it shouldn't be very different for other platforms.

# Setting Up Hugo

I use Homebrew, so installing Hugo was a breath

```
brew install hugo
```

Just to double check it's working:

```
hugo version
```

# Creating New Website

Time to create my website, I went ahead and used the following command:

```
hugo new site personal_website -f yml
```

This command creates a folder named personal_website with the new Hugo site in it. The -f yml part forces yml format for config files. This makes it easier to configure the website, and to use my target theme, but it's not a requirement.

I then added my chosen theme [PaperMod](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation) to the site:

```
git clone https://github.com/adityatelange/hugo-PaperMod themes/PaperMod --depth=1
```

## Adding Content

Now comes the tedious part of the job, creating new pages and copying over the content, page by page. To create a new post, I used the following command:

```
hugo new posts/my-first-post.md
```

Then I went ahead and copied the content of the first post in my original WordPress blog, pasted it in the newly created .md page, and reformatted it with MarkDown format. Note that by default, Hugo (as of now) uses [Goldmark](https://gohugo.io/getting-started/configuration-markup#goldmark).

This step took most of the time, as I needed to go back to my original website, look at the formatting of the original posts, and reapply it to the new website.

Now, programmers are very fond of automating things, so you might as me why I didn't try to automate the process? Well, mainly because there are less than 10 pages to convert, and the effort and time most likely wouldn't have been worth it.

## Testing Website

It's important to keep testing the website during development so I can see the progress. Hugo has the answer to that:

```
hugo server -D
```

Which runs a server to serve the website so I can take a look and review. It reloads changes live, which is very flexible, useful, and saves a lot of time.

## Customizing Pages

I modified the top part (called [front matter](https://gohugo.io/content-management/front-matter/)) of the .md pages to customize how they look (e.g.: thumbnail, title... etc.). It was important to use the similar post dates as the original ones to keep the original order. Also I had to set _draft: false_ so I can publish the content.

That's it. Once I was done with adding and customizing content, it was time to publish the website!

# Publishing Website

To get the content ready for publishing, I deleted the _public_ folder created by Hugo's development server, and ran the following command to re-generate the website:

```
hugo server
```

Followed by:

```
hugo
```

Now I had the _release_ build of my website ready for publishing. Hugo creates a static website, so you can host it anywhere really. But I checked the popular hosting and automated deployment options [here](https://gohugo.io/hosting-and-deployment/), and found out that [hosting on Render](https://gohugo.io/hosting-and-deployment/hosting-on-render/) sounds great. It has CDN, auto deploys from GitHub (where I already host my website's repo), and even has SSL.

## Leaving WordPress

It was time to say goodbye now. I added my original domain as a [custom domain](https://render.com/docs/custom-domains) to render, and set up my WordPress DNS Records to point to Render according to [those instructions](https://render.com/docs/configure-namecheap-dns).

After a short while, I was able to browse my new website without a trouble.