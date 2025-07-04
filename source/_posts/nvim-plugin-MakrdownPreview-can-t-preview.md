---
title: 'nvim plugin - MakrdownPreview: can''t preview'
date: 2025-07-04 22:50:37
tags: 
  - nvim
---

## Env:

Nvim: v0.11.2 & Lazyvim

Terminal: Kitty

OS: Arch Linux

## How did I do

Create a .lua file in the following directory: `~/.config/nvim/lua/plugins/markdown-preview.lua`

Then add the following code:
```
return {
	"iamcco/markdown-preview.nvim",
	cmd = { "MarkdownPreviewToggle", "MarkdownPreview", "MarkdownPreviewStop" },
	ft = { "markdown" },
	build = function()
		vim.fn["mkdp#util#install"]()
	end,
}
```

Reopen nvim and run: `Lazy sync`

Edit a markdown file:
```markdown
# Hello
## World
![remote-img](https://www.rust-lang.org/logos/rust-logo-512x512.png)
```

I got this message: `Nodejs v24.0.2` after run this command in nvim: `MarkdownPreview`

Search in MarkdownPreview repo and found this issue: https://github.com/iamcco/markdown-preview.nvim/issues/695

I tried like this issue and it worked!
