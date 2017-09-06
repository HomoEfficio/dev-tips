# Jenkins 에서 외부 서버에 터미널로 붙어서 명령을 실행하기

Job의 구성 > Build > Execute shell 에서 다음과 같이 접속 및 명령 실행

```
ssh -i ~/.ssh/recopick.pem ubuntu@batch.recopick.com "cd /mnt/ubuntu/recopick/ci_jobs/monthly/item_clickratio_table_partition; python partition.py"
```
