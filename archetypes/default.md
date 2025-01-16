+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
+++

## Dependencies 
I'm an `archlinux` user so dependencies are for `arch` but it should work with any `nix` distribution.
- [Linux](https://wiki.archlinux.org/title/Installation_guide)
- [Golang](https://go.dev/doc/install)