## AWS EC2 인스턴스 내 도커에 구성된 DB(PostgreSQL)를 <br> 로컬PC(MacBook)에 자동 백업하는 과정
* EC2 인스턴스, 도커, 데이터베이스는 구성이 된 상태라고 가정함(아래 정보로)<br><br>
* sudo 권한이 필요한 작업이 있을 수 있음<br><br>
* EC2 인스턴스<br>
 - 계정명 : ec2-user<br>
 - 호스트 : x.xx.xx.xxx(Public IP OR Public DNS)<br>
 - PEM : my_key_pair.pem
<br><br>
* 도커<br>
 - 컨테이너 이름 : my_container
<br><br>
* 데이터베이스(PostgreSQL)<br>
 - 유저 : postgres<br>
 - 비번 : 1234<br>
 - DB명 : my_db<br>

---
1. DB 백업 쉘스크립트 작성(EC2) - /home/ec2-user/postgres_backup.sh

```
# postgres_backup.sh
# 백업 디렉토리 경로 및 백업 파일명 설정
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/home/ec2-user/backups/postgres
BACKUP_FILE=$BACKUP_DIR/postgres_backup_$DATE.sql

# 백업 디렉토리 생성
mkdir -p $BACKUP_DIR

# DB 백업
docker exec my_container pg_dumpall -U postgres > $BACKUP_FILE

# 이전 백업 파일 제거(7일 이상 된 파일 삭제)
find $BACKUP_DIR -type f -mtime +7 -name "*.sql" -exec rm {}\;
```

2. 백업 스케줄 설정(EC2)
```
# 매일 오전 2시에 실행됨
crontab -e 0 2 * * * /home/ec2-user/postgres_backup.sh
```

3. EC2 인스턴스에 백업된 sql파일 로컬PC로 복사(로컬) - ec2_to_local_backup.sh
```
# ec2_to_local_backup.sh
# EC2 인스턴스 정보
EC2_USER=ec2-user
EC2_HOST=x.xx.xx.xxx
EC2_BACKUP_DIR=/home/ec2-user/backups/postgres

# 로컬 정보
LOCAL_BACKUP_DIR=/backups/postgres
PEM_FILE=/keys/my_key_pair.pem

# 백업 디렉토리 생성
mkdir -p $LOCAL_BACKUP_DIR

# EC2 인스턴스에서 로컬로 백업 파일 복사
scp -i $PEM_FILE $EC2_USER@$EC2_HOST:$EC2_BACKUP_DIR/postgres_backup_*.sql $LOCAL_BACKUP_DIR
```

4. PEM파일 권한 변경(사용자만 읽을 수 있도록)
```
chmod 400 my_key_pair.pem
```

5. ec2_to_local_backup.sh 실행 스케줄 설정(로컬)
```
# 매일 오전 3시에 실행됨
crontab -e 0 3 * * * ec2_to_local_backup.sh
```



