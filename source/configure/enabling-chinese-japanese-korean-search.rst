..  _i18n:

Chinese, Japanese, and Korean Search
======================================

Enabling search for Chinese, Japanese and Korean (CJK) requires special configuration, since these languages do not contain spaces.

- See `database requirements documentation <https://docs.mattermost.com/install/requirements.html#database-software>`__ for how to setup search for these languages.

.. contents::
    :backlinks: top

Below is additional information on how to configure the database for different languages.

中文 / Chinese
--------------

数据库版本请参考： `配置要求 <https://docs.mattermost.com/install/requirements.html#database-software>`__ 。
其中MySQL的ngram配置可以参考 `Cannot search CJK contents <https://github.com/mattermost/mattermost-server/issues/2033#issuecomment-182336690>`__ 。

更多中文相关问题讨论请访问 `中文讨论组 <https://forum.mattermost.org/c/international/chinese>`__ 。

以Ubuntu 14.04 PosgreSQL 9.3 数据库 mattermost 为例。

编译scws
~~~~~~~~

.. code:: bash

    wget -q -O - http://www.xunsearch.com/scws/down/scws-1.2.2.tar.bz2 | tar xjf -
    cd scws-1.2.2
    ./configure
    make install

编译zhparser
~~~~~~~~~~~~

.. code:: bash

    sudo apt-get install --yes postgresql-server-dev-9.3 libpq-dev
    git clone https://github.com/amutu/zhparser.git
    SCWS_HOME=/usr/local make && make install

.. note::
  通过 Docker 镜像（mattermost/mattermost-prod-db）应用数据库的用户，请按照下述方法安装依赖项。

  .. code:: bash

    # Alpine 通过 apk add 命令安装依赖项
    apk add wget tar gcc git postgresql-dev 

创建extension以及增加解析配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: bash

   sudo -i -u postgres
   psql mattermost -c 'CREATE EXTENSION zhparser'
   psql mattermost -c 'CREATE TEXT SEARCH CONFIGURATION simple_zh_cfg (PARSER = zhparser);'
   psql mattermost -c 'ALTER TEXT SEARCH CONFIGURATION simple_zh_cfg ADD MAPPING FOR n,v,a,i,e,l WITH simple;'

设置postgresql
~~~~~~~~~~~~~~

将 /etc/postgresql/9.3/main/postgresql.conf 中 default_text_search_config 的值更改为 simple_zh_cfg，然后重启postgresql: sudo service postgresql restart

调试
~~~~

可以打开 mattermost 的配置 config/config.json 中 SqlSettings 的设置 Trace: true，然后可以在mattermost的标准输出看到执行的SQL语句。

.. code:: sql

    SELECT to_tsvector('simple_zh_cfg', '开始全面整修道路');
    SELECT to_tsvector('simple_zh_cfg', '开始全面整修道路') @@ to_tsquery('simple_zh_cfg', '全面');
    SELECT * FROM Posts WHERE Message @@ to_tsquery('simple_zh_cfg', '全面');

日本語 / Japanese
-----------------

日本語翻訳の改善は大歓迎です。自由に変更していただいて結構です。

検索設定
~~~~~~~~~

Mattermost で日本語検索をするためにはデータベースの設定変更が必要です

- `MySQL <https://docs.mattermost.com/install/requirements.html#database-software>`__

- `Postgres <https://github.com/mattermost/mattermost-server/issues/2159#issuecomment-206444074>`__

日本語(CJK)検索設定のドキュメントの改善にご協力ください

ガイド
~~~~~~

Qiita上で Mattermost のインストールおよび構成のガイドを提供しています。詳細については、`こちら <http://qiita.com/tags/Mattermost>`_ をご覧ください。

한국어 / Korean
---------------

이 문제에 대한 논의는 이 `이슈 <https://github.com/mattermost/mattermost-server/issues/2033>`_ 에서 시작되었습니다.

한국어 버전 이용 시 문제점을 발견하면 `Localization 채널 <https://community.mattermost.com/core/channels/localization>`__ 또는 `한국어 채널 <https://community.mattermost.com/core/channels/i18n-korean>`__ 에서 의견을 제시할 수 있습니다.

검색을 위한 데이터베이스 설정
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

PostgreSQL: PostgreSQL 데이터베이스에서는 따로 설정이 필요하지 않습니다.

MySQL: MySQL에서는 전문 검색(Full-text search) 기능에 제한이 있기 때문에 추가적인 작업이 필요합니다.

MySQL 해결 방법
~~~~~~~~~~~~~~~~~

1. `n-gram parser <https://mysqlserverteam.com/innodb-%EC%A0%84%EB%AC%B8-%EA%B2%80%EC%83%89-n-gram-parser/>`__ 를 이용하기 위해서는 MySQL의 버전이 5.7.6 이상이어야 합니다.

2. MySQL의 구성 파일에서 n-gram의 최소 토큰 크기를 다음과 같이 설정합니다.

.. code:: sql

    [mysqld]
    ft_min_word_len = 2
    innodb_ft_min_word_len = 2

3. 데이터베이스를 재시작합니다. (이 과정은 반드시 필요합니다.)

4. 일부 테이블의 전문 검색 색인을 다음과 같이 재구성합니다.

- 게시물 검색을 위한 설정 ( `참조 <https://github.com/mattermost/mattermost-server/issues/2033#issuecomment-182336690>`__ )

.. code:: sql

    DROP INDEX idx_posts_message_txt ON Posts;
    CREATE FULLTEXT INDEX idx_posts_message_txt ON Posts (Message) WITH PARSER ngram;

- 해시 태그 검색을 위한 설정 ( `참조 <https://github.com/mattermost/mattermost-server/pull/4555>`__ )

.. code:: sql

    DROP INDEX idx_posts_hashtags_txt ON Posts;
    CREATE FULLTEXT INDEX idx_posts_hashtags_txt ON Posts (Hashtags) WITH PARSER ngram;

- 사용자 검색을 위한 설정

  ``Users.idx_users_txt_all`` 과 ``Users.idx_users_names_all`` 을 n-gram 없이 재구성합니다.
