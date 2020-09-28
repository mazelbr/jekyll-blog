---
layout: post
title:  "Set up Jekyll and Github Pages"
tag: jekyll blog welcome
---

# Installing Jekyll for Blogging with GitHub Pages

## Installing Jekyll
The first step is to install Jekyll, which is based on Ruby. The easies way is to just follow the instructions on the [website](https://jekyllrb.com/docs/installation/windows/).

For my embaressment I am using Windows right now so I chose the respective guidline.
After the installation is complete we want to install `jekyll`. Ruby uses `gem` as the package manager so we install `jekyll` with `gem install jekyll bundler`. That should be it and we can check the installation with `jekyll -v`

## Create a new jekyll project (aka blog)
To create the file structure for the blog run `jekyll new <projectname>` for my purpose I choose *jekyll-blog* as the name. Jekyll then creates a folder named `jekyll-blog` with the following content among others:
```
jekyll-blog
----_config.yml
----_posts
----_sites
----Gemfile
```
The `config.yml` file is the setup file where you can specify the title, description and author among others. Furthermore you can state there which template you want to use.

**The `sides` directory is haneled by jekyll and should therefore not be touched.** (Unless you really know what you are doing)


## Get your Blog up and running
To get your blog up and running simply type `bundle exec jekyll serve` into the command line. This will set up a server at a port on which it can be accessed in the browser.

## Write Conent

The `_posts` folder is where your blockposts go, all markdown and html files in there will show up in your block. **Note** that jekyll expects the filename to be `YYYY-MM-DD-Title.md` and that your file should (at lead for markdown) start with 
```
---
layout: post
title:  "Welcome to Jekyll!"
tag: tag1 tag2
---
```
Other attributes could also be added like e.g. *author* or *categories*.

We will skip all the design choices, templates and so on, but do not want to be unmentioned that one can create draft for blockposts that should not be published jet.
## Drafts
Simply create a `draft` folder. All the files in there won't show up in your running instance. If you want to get a preview run the server with the `--drafts` extension
`jekyll `

## Publish your Blog in Github Pages
The first step is to create a new repository on github and make sure that it **does not include a README.md file**.

Open the `config.yml`file and update the `base_url` with the name of the repository that you just created. In my case `jekyll-blog`
`baseurl: "jekyll-blog"`
Then go to the Gemfile and uncomment the `github-pages` gem. Apparently one should comment out the `jekyll` gem but for me it worked by just uncommenting the version. Eventuall one has tu run `bundle update` or `bundle install` before it works.
```
gem "jekyll"#, "~> 4.1.1"
# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem "minima", "~> 2.5"
# If you want to use GitHub Pages, remove the "gem "jekyll"" above and
# uncomment the line below. To upgrade, run `bundle update github-pages`.
gem "github-pages", group: :jekyll_plugins
```

Then we use git to manage the rest. First of all we initialize a git repository and then create a branch called `gh-pages`. This is necessary since Github expects all the files for the page to be in that branch
```
git init
git checkout -b gh-pages
git add .
git remote add origin <repository url>
git push origin gh-pages
```