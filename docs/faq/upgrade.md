# 升级 常见问题

!!! question "下载 Docker 镜像很慢"
    ```sh
    cat /etc/docker/daemon.json
    ```
    ```yaml
    {
      "registry-mirrors": [
        "https://hub-mirror.c.163.com",
        "https://bmtrgdvx.mirror.aliyuncs.com",
        "http://f1361db2.m.daocloud.io"
      ],
      "log-driver": "json-file",
      "log-opts": {
        "max-file": "3",
        "max-size": "10m"
      }
    }
    ```
    可以使用其他的镜像源, 推荐使用阿里云的镜像源  _[申请地址](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)_

!!! question "导入数据库报错"
    ```sh
    ./jmsctl.sh restore_db /opt/jumpserver.sql
    ```
    ```vim hl_lines="3"
    开始还原数据库: /opt/jumpserver.sql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    ERROR 1022 (23000) at line 1237: Can't write; duplicate key in table 'xxxxxx'
    ERRO[0008] error waiting for container: context canceled
    read unix @->ar/run/docker.sock: read: connection reset by peer
    数据库恢复失败,请检查数据库文件是否完整，或尝试手动恢复！
    ```
    ```sh
    ./jmsctl.sh stop
    ```
    ```sh
    docker exec -it jms_mysql /bin/bash
    ```
    ```sh
    mysql -uroot -p$MYSQL_ROOT_PASSWORD
    ```
    ```mysql
    drop database jumpserver;
    create database jumpserver default charset 'utf8' collate 'utf8_bin';
    exit;
    exit
    ```
    ```sh
    ./jmsctl.sh restore_db /opt/jumpserver.sql
    # 注意: 确定在导入数据库的过程中没有错误
    ```
    ```sh
    ./jmsctl.sh start
    ```

!!! question "启动 jms_core 报错"
    ```sh
    ./jmsctl.sh start
    ```
    ```vim hl_lines="5-9"
    Creating network "jms_net" with driver "bridge"
    Creating jms_mysql ... done
    Creating jms_redis ... done
    Creating jms_core  ... done
    ERROR: for celery  Container "76b2e315f69d" is unhealthy.
    ERROR: for lina  Container "76b2e315f69d" is unhealthy.
    ERROR: for luna  Container "76b2e315f69d" is unhealthy.
    ERROR: for guacamole  Container "76b2e315f69d" is unhealthy.
    ERROR: for koko  Container "76b2e315f69d" is unhealthy.
    ERROR: Encountered errors while bringing up the project.
    ```
    ```sh
    docker logs -f jms_core --tail 200  # 如果没有报错就等表结构合并完毕后然后重新 start 即可
    ./jmsctl.sh start
    ```

!!! question "Table 'xxxxxx' already exists"
    ```sh
    ./jmsctl.sh stop
    ```
    ```sh
    if grep -q 'CHARSET=utf8;' /opt/jumpserver.sql; then
        cp /opt/jumpserver.sql /opt/jumpserver_bak.sql
        sed -i 's@CHARSET=utf8;@CHARSET=utf8 COLLATE=utf8_bin;@g' /opt/jumpserver.sql
    else
        echo "备份数据库字符集正确";
    fi
    ```
    ```sh
    docker exec -it jms_mysql /bin/bash
    ```
    ```sh
    mysql -uroot -p$MYSQL_ROOT_PASSWORD
    ```
    ```mysql
    drop database jumpserver;
    create database jumpserver default charset 'utf8' collate 'utf8_bin';
    exit;
    exit
    ```
    ```sh
    ./jmsctl.sh restore_db /opt/jumpserver.sql
    # 注意: 确定在导入数据库的过程中没有错误
    ```
    ```sh
    ./jmsctl.sh start
    ```

!!! question "kombu.exceptions.OperationalError"
    ```vim
    # Redis < 5.0.0 导致, 请更新 Redis 版本
    kombu.exceptions.OperationalError:
    Cannot route message for exchange 'ansible': Table empty or key no longer exists.
    Probably the key ('_kombu.binding.ansible') has been removed from the Redis databa
    ```

!!! question "Internal Server Error"
    ```sh
    docker logs -f jms_core --tail 200
    # 查看是否有报错，如果没有或者不完整请进入容器查看日志
    ```
    ```sh
    docker exec -it jms_core /bin/bash
    cat logs/jumpserver.log
    # 根据报错处理
    ```

!!! question "修改对外访问端口"
    ```sh
    vi /opt/jumpserver/config/config.txt
    ```
    ```vim hl_lines="23"
    # 说明
    #### 这是项目总的配置文件, 会作为环境变量加载到各个容器中
    #### 格式必须是 KEY=VALUE 不能有空格等

    # Compose项目设置
    COMPOSE_PROJECT_NAME=jms
    COMPOSE_HTTP_TIMEOUT=3600
    DOCKER_CLIENT_TIMEOUT=3600
    DOCKER_SUBNET=192.168.250.0/24

    ## IPV6
    DOCKER_SUBNET_IPV6=2001:db8:10::/64
    USE_IPV6=0

    ### 持久化目录, 安装启动后不能再修改, 除非移动原来的持久化到新的位置
    VOLUME_DIR=/opt/jumpserver

    ## 是否使用外部MYSQL和REDIS
    USE_EXTERNAL_MYSQL=0
    USE_EXTERNAL_REDIS=0

    ## Nginx 配置，这个Nginx是用来分发路径到不同的服务
    HTTP_PORT=80           # 默认单节点对外 http  端口  (*)
    HTTPS_PORT=8443
    SSH_PORT=2222

    ## LB 配置, 这个Nginx是HA时可以启动负载均衡到不同的主机
    USE_LB=0
    LB_HTTP_PORT=80         
    LB_HTTPS_PORT=443
    LB_SSH_PORT=2223

    ## Task 配置
    USE_TASK=1

    ## XPack
    USE_XPACK=0


    # Koko配置
    CORE_HOST=http://core:8080
    ENABLE_PROXY_PROTOCOL=true


    # Core 配置
    ### 启动后不能再修改，否则密码等等信息无法解密
    SECRET_KEY=
    BOOTSTRAP_TOKEN=
    LOG_LEVEL=INFO
    # SESSION_COOKIE_AGE=86400
    # SESSION_EXPIRE_AT_BROWSER_CLOSE=false

    ## MySQL数据库配置
    DB_ENGINE=mysql
    DB_HOST=mysql
    DB_PORT=3306
    DB_USER=root
    DB_PASSWORD=
    DB_NAME=jumpserver

    ## Redis配置
    REDIS_HOST=redis
    REDIS_PORT=6379
    REDIS_PASSWORD=

    ### Keycloak 配置方式
    ### AUTH_OPENID=true
    ### BASE_SITE_URL=https://jumpserver.company.com/
    ### AUTH_OPENID_SERVER_URL=https://keycloak.company.com/auth
    ### AUTH_OPENID_REALM_NAME=cmp
    ### AUTH_OPENID_CLIENT_ID=jumpserver
    ### AUTH_OPENID_CLIENT_SECRET=
    ### AUTH_OPENID_SHARE_SESSION=true
    ### AUTH_OPENID_IGNORE_SSL_VERIFICATION=true


    # Guacamole 配置
    JUMPSERVER_SERVER=http://core:8080
    JUMPSERVER_KEY_DIR=/config/guacamole/data/key/
    JUMPSERVER_RECORD_PATH=/config/guacamole/data/record/
    JUMPSERVER_DRIVE_PATH=/config/guacamole/data/drive/
    JUMPSERVER_ENABLE_DRIVE=true
    JUMPSERVER_CLEAR_DRIVE_SESSION=true
    JUMPSERVER_CLEAR_DRIVE_SCHEDULE=24

    # Mysql 容器配置
    MYSQL_ROOT_PASSWORD=
    MYSQL_DATABASE=jumpserver
    ```
    ```sh
    ./jmsctl.sh restart
    ```
