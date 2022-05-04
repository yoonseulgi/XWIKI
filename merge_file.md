# postgreSQL  
## Install postgreSQL(master, standby server)  
  - postgresql 설치
  ```shell
  sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm  
  sudo yum install -y postgresql14-server  
  ```
  - .bash_profile  
    - 간단하게 사용하고 싶은 postgresql의 명령어를 명시  
    ```text
    [ -f /etc/profile ] && source /etc/profile
    PGDATA=/var/lib/pgsql/14/data
    export PGDATA
    # If you want to customize your settings,
    # Use the file below. This is not overridden
    # by the RPMS.
    [ -f /var/lib/pgsql/.pgsql_profile ] && source /var/lib/pgsql/.pgsql_profile
    alias pgstart='/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data -l logfile start'
    alias pgrestart='/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data -l logfile restart'
    alias pgstatus='/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data -l logfile status'
    alias pgstop='/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data -l logfile stop'
    ```
  - initdb 설정하기  
    - 한글 정렬을 가능하게 하려면 initdb 설정이 필요  
    ```
    ```
