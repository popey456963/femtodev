+++
layout = "post"
title = "Blogging with Hugo & Travis CI"
date = "2018-01-12"
categories = ["Bash", "Linux"]
+++

I recently had the good fortune of being asked to work on a project with [Finnian Anderson](https://finnian.io/) and [Ben James](https://benjames.io/), the idea was rather a simple one, just requiring a blog to setup.  One of our concerns though was that all of us had done devops before and weren't particuarly keen to do it again, thus we needed a blog that had as little overhead as possible in terms of setup and maintenance time but still was fast to load and easy to use.  We're also don't have lots of money to waste on a side project, so it had to be as close to free as we could get it.

Our initial thought was [Medium](http://medium.com/), however that turned out to not be feasible since they [stopped offering custom domains](https://help.medium.com/hc/en-us/articles/115005579728-Custom-domains-on-Medium) a few months back.  This walls us in to their service, detracts from our image and discouraged us from using the service.

Hugo has been gaining popularity recently and I had an hour to spare, so we thought we'd try something new.  Unfortunately, a large proportion of guides on this subject seem out of date, hence the creation of this post.

### The Plan

```
+------+       +----------+
|      |       |          |
| USER | +---> | GH Pages | <-------------+
|      |       |          |               |
+------+       +----------+               +

+--------+       +----------+       +-----------+       +------+
|        |       |          |       |           |       |      |
| WRITER | +---> | Git Repo | +---> | Travis CI | <---> | Hugo |
|        |       |          |       |           |       |      |
+--------+       +----------+       +-----------+       +------+
```

As you can see from the flowchart above, the plan is to let Travis compile a selection of markdown files using Hugo when it detects a change in the Git repository. Travis can then use it's inbuilt "publish" module to then change the github pages content.

### Github

Although we will add data to the repository later, there are some initial items that need to be setup within Github:

1. Generate a new [Personal Access Token](https://github.com/settings/tokens) from Github.  This will be used by Travis to push the changes to Github, so it will need access to the `repo` scope.
2. Create a branch called `gh-pages`.  You can do this via the GUI on the main repository page like so:

![](https://puu.sh/yZQYO/210fe701a8.png)

3. If you so desire, you can setup a custom domain name for your repository by entering it at `https://github.com/:[user]/:[repo]/settings`.
4. On the same page, setup the "Source" for GitHub Pages to be the "gh-pages" branch.

### Hugo

Hugo itself seems exceptionally easy to use, for the purpose of this blog post our layout will follow the layout specific by the [Hugo example](https://github.com/gohugoio/hugoBasicExample).  This blog assumes that the output folder is set to `docs`, if any other folder name is selected it simply involves altering the two occurances of `docs` within the `.travis.yml` file.

```
repo.
│   config.toml
│
├───content
│   │   about.md
│   │
│   └───post
│           creating-a-new-theme.md
│           goisforlovers.md
│           hugoisforlovers.md
│           migrate-from-jekyll.md
│
├───layouts
│
└───static
```

> It's wise to check out your `config.toml` and alter the settings to your personal needs.  Of special note is that "baseurl" is often set in a number of example projects.  If your project is running from the root of a domain or sub-domain, it appears safe to leave this blank

Once you have setup the initial blog layout to your liking, you must also create another file called `.nojekyll` with no content.  This will tell Github that we are, in fact, not using jekyll and instead using our own platform.

### Travis CI

The final step is to get your blog compiled.  Login to Travis and visit your profile [here](https://travis-ci.org/profile).  Scroll down the list of repositories and enable it for your blog repository.

> You may need to press the "Sync account" button in the top-right to inform Travis you've created a new repository.  This can take a few minutes, maybe grab a coffee or do some exercise?

Once you have enabled it, you will need to add a `.travis.yml` file to your repository.  This lets Travis know what to do to run your project.  Our setup is fairly simple, here is the file I use:

```
# Install the apt prerequisites
addons:
  apt:
    packages:
      - python-pygments
      
before_install:
  - wget https://github.com/gohugoio/hugo/releases/download/v0.32.3/hugo_0.32.3_Linux-32bit.deb
  - sudo dpkg -i hugo_0.32.3_Linux-32bit.deb
  - rm hugo_0.32.3_Linux-32bit.deb
  
# Clean and don't fail
install:
  - rm -rf docs || exit 0

# Build the website
script:
  - hugo

# Deploy to GitHub pages
deploy:
  provider: pages
  skip_cleanup: true
  local_dir: "${TRAVIS_BUILD_DIR}/docs"
  github_token: $GITHUB_TOKEN # Set in travis-ci.org dashboard
  on:
    branch: master
```

Splitting this up into sections:

- **addons**: Any initial packages to be installed, here we use the `python-pygments` library to add code highlighting.
- **before_install**: Install Hugo using `dpkg`.  It might be suggested to grab the latest version from [here](https://github.com/gohugoio/hugo/releases).
- **install**: Remove any existing blog content, if `docs` does not exist it will error out, the `|| exit 0` section will catch that error so that it does not break the build.
- **script**: Run the `hugo` command, add any arguments you wish to compile with here, as well as other scripts you wish to run before `hugo`.
- **deploy**: Copy the `docs` folder to the `gh-pages` branch of Github if the current branch is the `master` branch (to stop a loop).

You will notice a `$GITHUB_TOKEN` reference in the deployment section, this is the `Personal Access Token` we saved earlier.  To add this securely to the build, we use Travis environment variables at `https://travis-ci.org/:[user]/:[repo]/settings`.  Just add a field with the name of `GITHUB_TOKEN` and a value of the token.  Note that this does not automatically save, you will need to click the `add` button on the far right.

Once this is done you should be able to add any content you wish to the blog and see it automatically shown at either the custom domain you defined, or `:[user].github.com/:[repo]/`.

### Finishing Up

Overall this experiment was a great success.  The entire system took barely any time to setup and all the different tools worked exceptionally well together.  The one drawback I've noticed so far with Hugo is that error messages are sometimes fairly cryptic, however it catches a lot of common mistakes I made when altering the configuration file and so those errors are rare to see.

Another advantage of this design is that every element is easily replaceable.  There are dozens of Git repositories that offer hosting services similar to Github pages, hundreds of continuous integration services and thousands of blogging software out there.  The only issue I forsee is that project layouts differ a lot, so some reorganisation might be necessary.  