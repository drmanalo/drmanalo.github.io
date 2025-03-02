+++
date = "{{ .Date }}"
draft = true
title = "{{ replace .File.ContentBaseName "-" " " | title }}"
+++

## Prerequisites
Being a `Macbook` user means dependencies are for `brew` but this blog post should work with any `unix` distribution.
- [homebrew](https://brew.sh/)
