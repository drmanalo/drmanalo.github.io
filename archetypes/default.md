+++
date = "{{ .Date }}"
draft = true
title = "{{ replace .File.ContentBaseName "-" " " | title }}"
+++

## Prerequisites 
Being an `archlinux` user means dependencies are for `arch` but this blog post should work with any `nix` distribution.
- [Linux](https://wiki.archlinux.org/title/Installation_guide)