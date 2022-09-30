---
title: "Film 0"
date: 2022-09-30T15:56:19+08:00
draft: false
tags: ["film"]
---
{{range .Resources.Match "images/*" }}
    <img src="{{ .RelPermalink }}" />
{{ end }}