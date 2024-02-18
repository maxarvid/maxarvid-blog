+++
title = "Continuous Deployment with Github Actions"
date = 2024-02-18
+++

It only takes two `nslookup`s to figure out that the url of this site sits behind an AWS Cloudfront distribution. Since this is a static site you could guess that the site lives in an AWS S3 bucket, and you would be right. The Zola docs provide convenient instructions for [how to deploy to an S3 bucket](https://www.getzola.org/documentation/deployment/aws-s3/), but as is often the case they cover the most general use-case. I followed these instructions, but ran into trouble as I was using a theme. 

There is a slight discrepancy in how the docs recommend you install a theme, and how they theme publishers direct you. The Zola docs suggest the following:
```sh
$ cd themes
$ git clone <theme repository URL>
```
Which as you can probably guess clones the files into your repository and you are done. I went for a different route thinking I would try to learn something about [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules), also in deference to the theme's author's recommendation:
```sh
$ cd themes/
$ git submodule add git://git.figbert.com/d3c3nt.git
```
Now that worked well too, until it came time to try out the Github Action to deploy to the S3 bucket. While I have all the files of the theme on my machine, in my Github repo the theme's directory is represented as a link (which I cannot click), with a reference to what I assume is the commit when I added the submodule. 

In addition to not me not being able to see the files on my Github's repo's GUI (by design), this means that any Github Action won't necessarily have access to the file either, unless you add a step that fetches them. And by steps I mean just add a `with` to when you are checking out the code in the runner:

```yml
- name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
```
Which did the trick! Full workflow file below for ease of reference:

```yml
name: Build and Publish to AWS
on:
  push:
    branches:
      - main
jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - uses: taiki-e/install-action@v2
        with:
          tool: zola@0.18.0
      - name: Build
        run: zola build
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-north-1
      - name: Publish to AWS S3
        run: aws s3 sync --delete public/ s3://${{ secrets.S3_BUCKET }}
      - name: Invalidate CloudFront cache
        run: aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```