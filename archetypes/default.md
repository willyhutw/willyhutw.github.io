---
date: {{ .Date }}
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
type: '{{ .Type }}'
draft: true
---
