# logrotate

- 유닉스 계열 OS에서 로그 파일을 로테이트 시켜주는 명령

## 실행 메커니즘

- logrotate는 데몬이 아닌 단순 실행파일이며, 따라서 주기적으로 반복 실행되게 하려면 cron 도움 필요
  - cron 에 등록된 명령은 보통 다음과 같이 `/etc/cron*`에서 확인 가능
    ```
    $ ll /etc/cron*
    -rw-r-----  1 root root   0 Jan 14  2022 /etc/cron.deny
    -rw-r--r--  1 root root 449 Mar 29  2023 /etc/crontab
    
    /etc/cron.d:
    total 16
    -rw-r--r-- 1 root root 126 Mar 29  2023 0hourly
    -rw-r--r-- 1 root root 366 Mar  4 10:40 cron-sastoolkit
    -rw-r--r-- 1 root root 108 Jan  8  2022 raid-check
    -rw------- 1 root root 233 Mar 29  2023 sysstat
    
    /etc/cron.daily:
    total 8
    -rwxr-xr-x 1 root root 795 Mar  4 10:40 anacron-daily
    -rwx------ 1 root root 219 Apr  1  2020 logrotate
    
    /etc/cron.hourly:
    total 4
    -rwxr-xr-x 1 root root 392 Jan 14  2022 0anacron
    
    /etc/cron.monthly:
    total 0
    
    /etc/cron.weekly:
    total 0
    ```
  - `/etc/cron.daily/logrotate` 내용을 확인해보면 결국 `/etc/logrotate.conf` 파일을 참조한다.
    ```
    $ sudo cat /etc/cron.daily/logrotate
    #!/bin/sh
    
    /usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
    EXITVALUE=$?
    if [ $EXITVALUE != 0 ]; then
        /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
    fi
    exit 0
    ```
  - `/etc/logrotate.conf` 안에 `include /etc/logrotate.d`라고 지정돼 있으므로, 결국 **`/etc/logrotate.d` 폴더 아래에 logrotate 설정 파일을 추가하면 daily 기반으로 실행된다**
    ```
    $ sudo cat /etc/logrotate.conf
    # see "man logrotate" for details
    # rotate log files weekly
    weekly
    
    # keep 4 weeks worth of backlogs
    rotate 4
    
    # create new (empty) log files after rotating old ones
    create
    
    # use date as a suffix of the rotated file
    dateext
    
    # uncomment this if you want your log files compressed
    #compress
    
    # RPM packages drop log rotation information into this directory
    include /etc/logrotate.d
    
    # no packages own wtmp and btmp -- we'll rotate them here
    /var/log/wtmp {
        monthly
        create 0664 root utmp
            minsize 1M
        rotate 1
    }
    
    /var/log/btmp {
        missingok
        monthly
        create 0600 root utmp
        rotate 1
    }
    
    # system-specific logs may be also be configured here.
    ```

- 더 자세한 내용은 https://blog.o3g.org/server/logrotate를-활용하여-로그-관리하기/ 참고    

## 설정
- `/etc/logrotate.d/nginx` 파일에 다음과 같이 작성하면 된다
  - 설정 옵션은 man 페이지에 잘 나와있는데 openresty nginx 라면 대략 다음과 같이 지정하면 된다
    ```
    /usr/local/openresty/nginx/logs/*log {
        daily
        dateext # 아카이브 된 파일명에 -yyyyMMdd 추가
        dateyesterday # 아카이브 된 파일명에 사용되는 날짜를 아카이브 수행 날짜가 아닌 전날(실제 로그가 적재된 날짜) 날짜 사용
        missingok
        notifempty
        rotate 30
        compress
        delaycompress
        copytruncate
        create 600 root root
        postrotate # logrotate 작업 이후 아카이브된 파일 말고 새 파일에 로그 적재
            /bin/kill -USR1 `cat /usr/local/openresty/nginx/logs/nginx.pid 2>/dev/null` 2>/dev/null || true
        endscript
    }
    ```

## 실행

- 먼저 디버그 옵션인 `-d`를 사용해서 `/usr/sbin/logorate -d /etc/logrotate.d/nginx`를 실행하면 다음과 같이 미리 오류 여부를 확인할 수 있다.
  ```
  $ /usr/sbin/logrotate -d /etc/logrotate.d/nginx
  reading config file /etc/logrotate.d/nginx
  Allocating hash table for state file, size 15360 B
  
  Handling 1 logs
  
  rotating pattern: /usr/local/openresty/nginx/logs/*log  after 1 days (30 rotations)
  empty log files are not rotated, old logs are removed
  considering log /usr/local/openresty/nginx/logs/access.log
    log does not need rotating (log has been already rotated)considering log /usr/local/openresty/nginx/logs/error.log
    log does not need rotating (log has been already rotated)considering log /usr/local/openresty/nginx/logs/host.access.log
    log does not need rotating (log has been already rotated)
  ```
- `/etc/logrotate.d` 폴더에 추가해두면 cron 에 의해 다음날 실행되지만 지금 바로 실행하고 싶다면 `-f` 옵션을 사용해서 `/usr/sbin/logrotate -f /etc/logrotate.d/nginx`를 실행하면 된다.

