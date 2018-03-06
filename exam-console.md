## Exam Console

### bash

```bash
apt-get install git

cd $HOME
git clone https://github.com/jonmosco/kube-ps1.git
git clone https://github.com/ahmetb/kubectx.git

cat >> ~/.bashrc <<EOF
source <(kubectl completion bash)
PATH="\$HOME/kubectx:\$PATH"
source "\$HOME/kube-ps1/kube-ps1.sh"
EOF
echo 'PS1="[\u@\h \W \$(kube_ps1)]\$ "' >> ~/.bashrc

source ~/.bashrc
```

### tmux

```bash
apt-get install tmux

cat > ~/.tmux.conf <<EOF
unbind C-b
set -g prefix C-q

setw -g mode-keys vi

unbind %
bind-key s split-window -v -c "#{pane_current_path}"
unbind '"'
bind-key v split-window -h -c "#{pane_current_path}"
unbind c
bind-key c new-window -c "#{pane_current_path}"

bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

## copy & paste
bind y copy-mode
bind p paste-buffer

## colors
set-option -g default-terminal screen-256color

set -g status-fg white
set -g status-bg black

setw -g window-status-fg cyan
setw -g window-status-bg default
setw -g window-status-attr dim
setw -g window-status-current-fg white
setw -g window-status-current-bg red
setw -g window-status-current-attr bright

set -g pane-border-fg green
set -g pane-border-bg black
set -g pane-active-border-fg white
set -g pane-active-border-bg yellow
EOF

tmux
```
