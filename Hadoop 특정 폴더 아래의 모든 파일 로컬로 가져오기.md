```shell
#!/usr/bin/env bash

# $1: YYYY-MM-DD
# $2: TARGET_DIR
# 참고: S3에서 HDFS로 받아올 때 2017-08-17/15~23 디렉토리에는 2017-08-16/15~23의 파일이 다운로드 되어 >있음.
# 즉 YYYY-MM-DD 디렉토리 아래의 데이터는 모두 KST 기준으로 되어 있으므로, HDFS에서 가져올 때는 날짜만 >맞추면 KST 기준 그 날짜의 데이터를 받게됨
if [ "$1" == "" ] || [ "$2" == "" ]; then
        echo 'Usage: ./get_files_from_hdfs.sh YYYY-MM-DD TARGET_DIR/'
        exit 0
fi

SRC_DIR=/user/ubuntu/coruscant/export/click-log/itemitem_click_log/$1
TARGET_DIR=$2
mkdir ${TARGET_DIR}

function getHdfsFiles() {
        for i in {0..23}
        do
                if [ $i -le 9 ]; then
                        i="0${i}"
                fi
                hdfs dfs -get ${SRC_DIR}/$i ${TARGET_DIR} || return 1
        done

        echo 'SUCCESS!'
}

getHdfsFiles || exit 1
exit 0
```
