title: '[VIM]plug coc.nvim'
date: '2024-10-03 15:55:44'
updated: '2024-10-03 15:55:48'
tags:
  - vim
---
在coc.nvim更新到0.0.82之后，shortcut映射方式改变。

.vimrc配置如下：

```bash
function! CheckBackSpace() abort
  let col = col('.') - 1
  return !col || getline('.')[col - 1]  =~ '\s'
endfunction
inoremap <silent><expr> <TAB>
      \ coc#pum#visible() ? coc#pum#next(1):
      \ CheckBackSpace() ? "\<Tab>" :
      \ coc#refresh()
inoremap <expr><S-TAB> coc#pum#visible() ? coc#pum#prev(1) : "\<C-h>"

" below is for using ENTER for completion, I actually don't like it, CTRL+Y works better for me, you can omit this part if you are like me

inoremap <silent><expr> <cr> coc#pum#visible() && coc#pum#info()['index'] != -1 ? coc#pum#confirm() :
        \ "\<C-g>u\<CR>\<c-r>=coc#on_enter()\<CR>"
```