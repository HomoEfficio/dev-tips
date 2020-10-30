# Tmux 설치 및 사용 on Mac

## 설치

>brew install tmux

```
...
==> tmux
Example configuration has been installed to:
  /usr/local/opt/tmux/share/tmux

Bash completion has been installed to:
  /usr/local/etc/bash_completion.d
```

## 개념

- session: 여러 윈도우를 가지는 단위
- window: 하나의 세션 내에서 사용되는 여러 창 단위
- pane: 하나의 윈도우 내에서 사용되는 여러 실제 입력 창 단위

복잡하게 생각할 것 없이 그냥 하나의 세션, 하나의 윈도우에 pane 만 추가/삭제/분할 해서 사용

## 명령

가장 단순하게 아래 명령만 알면 쓰는 데 별 지장 없다.

```
# 세션 실행
tmux: tmux new -s 0 -n 어쩌구 과 같다.
tmux new -s <session-name> -n <window-name>

# pane 분할
[ctrl + b], %: 세로 분할 (열 추가)
[ctrl + b], ": 가로 분할 (행 추가)

# pane 이동
[ctrl + b], q: 번호 지정으로 이동
[ctrl + b]. o: 차례로 이동

# pane 제거
[ctrl + d] 또는 exit 입력
마지막 pane이 제거되면 윈도우와 세션도 모두 제거 된다.

# pane 크기 조절
[ctrl + b], :, resize-pane -LRDU 5

# pane 레이아웃 변경
[ctrl + b], spacebar

# 여러 pane에 동시 입력/해제
설치 시 제공되는 /usr/local/opt/tmux/share/tmux/example_tmux.conf 내용을 참고해서,
~/.tmux.conf 파일에 다음과 같이 입력

bind y set synchronize-panes\; display 'synchronize-panes #{?synchronize-panes,on,off}'

이후 [ctrl + b], y 를 누르면 동시 입력 모드 활성화/해제 가 toggle 된다.
```
