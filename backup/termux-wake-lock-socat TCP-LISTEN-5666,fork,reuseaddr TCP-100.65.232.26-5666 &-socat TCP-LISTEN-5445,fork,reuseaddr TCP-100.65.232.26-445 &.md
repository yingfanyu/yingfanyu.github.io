tmux new-session -d -s port-forward "termux-wake-lock; socat TCP-LISTEN:5666,fork,reuseaddr TCP:100.65.232.26:5666 & socat TCP-LISTEN:5445,fork,reuseaddr - [ ] TCP:100.65.232.26:445 & wait"
