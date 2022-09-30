---
title: "Film 0"
date: 2022-09-30T15:56:19+08:00
draft: false
tags: ["film"]
---
{{ .Resources.ByType "image" }}
    <img src="data:{{ .MediaType }};base64,{{ .Content | base64Encode }}">
{{ end }}