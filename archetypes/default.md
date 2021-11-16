---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
archives:
    - {{ now.Format "2006-01" }}
categories: ["blog"]
tags: ["note", "unreal"]
draft: true
---

