```
[alias]
        co = checkout
        ci = commit
        st = status
        unstage = reset HEAD --
        ciam = !git add . && git commit -m
        cim = commit -m
        lme = log --author='Homo Efficio' --graph --all --pretty=format:'%C(yellow)%h%C(cyan)%d%Creset %s %C(white)- %an, %ar%Creset'
        fod = fetch origin develop
        fom = fetch origin master
        rbod = rebase origin/develop
        rbom = rebase origin/master
        rb = rebase
        psod = push origin develop
        psom = push origin master
```
