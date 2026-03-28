let mapleader = ' '
let g:mapleader = ' '

syntax on

set history=2000

filetype on
filetype plugin on
filetype indent on

set nocompatible                
set autoread                    
set shortmess=atI

set magic                       
set title                       
set nobackup                    

set novisualbell                
set noerrorbells                
set visualbell t_vb=            
set t_vb=
set tm=500

set scrolloff=7                 

set ruler                       
set number                      
set relativenumber              
set nowrap
set showcmd                     
set showmode                    
set showmatch                   
set matchtime=2                 

set hlsearch
set incsearch
set ignorecase
set smartcase

set expandtab
set smarttab
set shiftround

set autoindent smartindent shiftround
set shiftwidth=4
set tabstop=4
set softtabstop=4
nnoremap <leader>hl :nohlsearch<CR>

set foldenable
set foldmethod=indent
set foldlevel=99
let g:FoldMethod = 0
map <leader>zz :call ToggleFold()<cr>
fun! ToggleFold()
    if g:FoldMethod == 0
        exe "normal! zM"
        let g:FoldMethod = 1
    else
        exe "normal! zR"
        let g:FoldMethod = 0
    endif
endfun

set encoding=utf-8
set fileencodings=ucs-bom,utf-8,cp936,gb18030,big5,euc-jp,euc-kr,latin1
set termencoding=utf-8
set ffs=unix,dos,mac
set formatoptions+=m
set formatoptions+=B

set selection=inclusive
set selectmode=mouse,key

set completeopt=longest,menu
set wildmenu
set wildmode=longest,list,full
set wildignore=*.o,*~,*.pyc,*.class

set backspace=indent,eol,start
set whichwrap+=<,>,h,l

if has("autocmd")
  au BufReadPost * if line("'\"") > 1 && line("'\"") <= line("$") | exe "normal! g'\"" | endif
endif

" ============================ theme and status line ============================

set background=dark
colorscheme desert

hi! link SignColumn   LineNr
hi! link ShowMarksHLl DiffAdd
hi! link ShowMarksHLu DiffChange

set statusline=%<%f\ %h%m%r%=%k[%{(&fenc==\"\")?&enc:&fenc}%{(&bomb?\",BOM\":\"\")}]\ %-14.(%l,%c%V%)\ %P
set laststatus=2

" ============================ specific file type ===========================

autocmd FileType python set tabstop=4 shiftwidth=4 expandtab ai
autocmd FileType ruby set tabstop=2 shiftwidth=2 softtabstop=2 expandtab ai
autocmd BufRead,BufNew *.md,*.mkd,*.markdown  set filetype=markdown.mkd

autocmd BufNewFile *.sh,*.py exec ":call AutoSetFileHead()"
function! AutoSetFileHead()
    if &filetype == 'sh'
        call setline(1, "\#!/bin/bash")
    endif

    if &filetype == 'python'
        call setline(1, "\#!/usr/bin/env python")
        call append(1, "\# encoding: utf-8")
    endif

    normal G
    normal o
    normal o
endfunc

autocmd FileType c,cpp,java,go,php,javascript,puppet,python,rust,twig,xml,yml,perl autocmd BufWritePre <buffer> :call <SID>StripTrailingWhitespaces()
fun! <SID>StripTrailingWhitespaces()
    let l = line(".")
    let c = col(".")
    %s/\s\+$//e
    call cursor(l, c)
endfun

" ============================ key map ============================

map <C-j> <C-W>j
map <C-k> <C-W>k
map <C-h> <C-W>h
map <C-l> <C-W>l

inoremap jk <Esc>

nnoremap <silent> n nzz
nnoremap <silent> N Nzz
nnoremap <silent> * *zz
nnoremap <silent> # #zz
nnoremap <silent> g* g*zz

noremap <silent><leader>/ :nohls<CR>

vnoremap < <gv
vnoremap > >gv

map Y y$
nnoremap y "+y
vnoremap y "+y
nnoremap yy "+yy

nnoremap ; :

cmap w!! w !sudo tee >/dev/null %
