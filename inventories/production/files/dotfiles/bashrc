# Enable bash programmable completion features in interactive shells
if [ -f /usr/share/bash-completion/bash_completion ]; then
	. /usr/share/bash-completion/bash_completion
elif [ -f /etc/bash_completion ]; then
	. /etc/bash_completion
fi

####################################################
####################################################

#History stuff--- Timeformat, ignore dupes, buffer size
HISTTIMEFORMAT="%F %T "
HISTCONTROL=ignoredups
HISTSIZE=2000
HISTFILESIZE=2000

# Check the window size after each command and, if necessary, update the values of LINES and COLUMNS
shopt -s checkwinsize

#bash appends to his, inbstead of overwriting it.  this keeps history across terminals
shopt -s histappend
PROMPT_COMMAND='history -a'

# Allow ctrl-S for history navigation (with ctrl-R)
stty -ixon

# Ignore case on auto-completion
# Note: bind used instead of sticking these in .inputrc
if [[ $iatest > 0 ]]; then bind "set completion-ignore-case on"; fi

# Show auto-completion list automatically, without double tab
if [[ $iatest > 0 ]]; then bind "set show-all-if-ambiguous On"; fi

# Set the default editor
export EDITOR=nvim
export VISUAL=nvim

# To have colors for ls and all grep commands such as grep, egrep and zgrep
export CLICOLOR=1
export LS_COLORS='no=00:fi=00:di=00;34:ln=01;36:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arj=01;31:*.taz=01;31:*.lzh=01;31:*.zip=01;31:*.z=01;31:*.Z=01;31:*.gz=01;31:*.bz2=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.jpg=01;35:*.jpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.avi=01;35:*.fli=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.ogg=01;35:*.mp3=01;35:*.wav=01;35:*.xml=00;31:'
alias grep="/usr/bin/grep $GREP_OPTIONS"
unset GREP_OPTIONS

# Color for manpages in less makes manpages a little easier to read
export LESS_TERMCAP_mb=$'\E[01;31m'
export LESS_TERMCAP_md=$'\E[01;31m'
export LESS_TERMCAP_me=$'\E[0m'
export LESS_TERMCAP_se=$'\E[0m'
export LESS_TERMCAP_so=$'\E[01;44;33m'
export LESS_TERMCAP_ue=$'\E[0m'
export LESS_TERMCAP_us=$'\E[01;32m'

#Colors
blk='\[\033[01;30m\]' # Black
red='\[\033[01;31m\]' # Red
grn='\[\033[01;32m\]' # Green
ylw='\[\033[01;33m\]' # Yellow
blu='\[\033[01;34m\]' # Blue
pur='\[\033[01;35m\]' # Purple
cyn='\[\033[01;36m\]' # Cyan
wht='\[\033[01;37m\]' # White
clr='\[\033[00m\]'    # Reset

####################################################
####################################################

#Alias's

alias ..='cd ..;pwd'
alias ...='cd ../..;pwd'
alias ....='cd ../../..;pwd'
alias c='clear'
alias h='history'
alias tree='tree --dirsfirst -F'
alias mkdir='mkdir -p -v'
alias sl='cd'

if command -v exa >/dev/null 2>&1; then
    alias la='exa -a --group-directories-first'
    alias ls='exa --group-directories-first'
    alias lla='exa -a -l --header --group-directories-first'
    alias ll='exa -l --header --group-directories-first'
    alias lg='exa -l -a --git --header --group-directories-first'
    alias lt='exa -a --tree --group-directories-first'
else
    alias la='ls -A'
    alias ls='ls --color=auto'
    alias lla='ls -lah'
    alias ll='ls -lh'
    alias lg='ls -lah'
    alias lt='ls -A'
fi

alias snvim='sudo -e'

alias addservice='python3 addservice.py'

#Functions
function hg() {
	history | grep "$1"
}

function find_largest_files() {
	du -h -x -s -- * | sort -r -h | head -20
}

function git_branch() {
	if [ -d .git ]; then
		printf "%s" "($(git branch 2>/dev/null | awk '/\*/{print $2}'))"
	fi
}

# Set the prompt.
bash_prompt() {
  PS1="${debian_chroot:+($debian_chroot)}${blu}\$(git_branch)${cyn}\u@\h${pur} \W${grn} > ${clr}"
}
# function bash_prompt() {
#   PS1='${debian_chroot:+($debian_chroot)}'${blu}'$(git_branch)'${ylw}'@\h'${pur}' \W'${grn}' > '${clr}
# }
bash_prompt
