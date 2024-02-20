# Fluentd

https://github.com/thatslifebro/MiniGameHeavenAPIServer �� ������Ʈ�� ������� �Ѵ�.

## fluentd.conf

1. json structured �α� ������ �о�´�.
1. Error �α׿� Info �α׷� �и��Ѵ�.
1. message �ʵ��� ���뿡�� api �ʵ带 �����Ѵ�.
1. Error �α׸� mysql�� �����Ѵ�.
1. Info �α׿��� api�� Login, MiniGameSave, �Ѵ� �ƴ� ��� 3������ �±׸� ���δ�.
1. �±׿� ���� �α׸� �и��Ͽ� mysql�� �����Ѵ�.

---
```conf
<source>
	@type tail
	path /App/log/*
	pos_file /tmp/fluent/temp.pos
	tag docker.log
	<parse>
		@type json
	</parse>
</source>
```
=> /App/log/ ��ο� �ִ� ��� ������ �о�� docker.log �±׸� �ٿ��ش�.
```conf
<match docker.log>
	@type rewrite_tag_filter
	<rule>
		key LogLevel
		pattern Error
		tag error.${tag}
	</rule>
	<rule>
		key LogLevel
		pattern Information
		tag info.${tag}
	</rule>
</match>
```
=> docker.log �±׸� ���� �α׸� �о�� LogLevel�� Error�� ��� error.docker.log �±׸� �ٿ��ְ�, Information�� ��� info.docker.log �±׸� �ٿ��ش�.
```conf
<filter info.docker.log>
	@type record_transformer
	enable_ruby
	<record>
		api ${record["Message"].split(']')[0].delete('[')}
	</record>
</filter>
```
=> info.docker.log �±׸� ���� �α׸� �о�� Message �ʵ带 �������� api �ʵ带 �������ش�.
```conf
<match error.docker.log>
	@type sql
	host db
	port 3306
	database log_db
	adapter mysql2
	username root
	password Rlatjd095980!
	<table>
		table error_log
		column_mapping 'LogLevel:level, Message:msg, Timestamp:timestamp'
	</table>
</match>
```
=> error.docker.log �±׸� ���� �α׸� log_db �����ͺ��̽��� error_log ���̺� �����Ѵ�.
```conf
<match info.docker.log>
	@type rewrite_tag_filter
	<rule>
		key api
		pattern Login
		tag login.${tag}
	</rule>
	<rule>
		key api
		pattern MiniGameSave
		tag game.${tag}
	</rule>
	<rule>
		key api
		pattern MiniGameLoad|Login
		invert true
		tag api.${tag}
	</rule>
</match>
```
=> info.docker.log �±׸� ���� �α׸� api�� Login, MiniGameSave, �� �� �ƴ� �ܿ� 3������ ������ �±׸� �ٿ��ش�.
```conf
<match api.**>
	@type sql
	host db
	port 3306
	database log_db
	adapter mysql2
	username root
	password Rlatjd095980!
	<table>
		table info_api_log
		column_mapping 'LogLevel:level, Timestamp:timestamp, api:api, uid:uid, result:result'
	</table>
</match>
```
=> api.** �±׸� ���� �α׸� log_db �����ͺ��̽��� info_api_log ���̺� �����Ѵ�.

LogLevel, Timestamp, uid, result�� json_structred �α׿� ���ԵǾ� �ֱ� ������ �ٷ� ����� �� �ִ�.

�������� �α׸� json ���·� �����ؾ� �����ϴ�.
```conf
<match login.**>
	@type copy

	<store>
		@type sql
		host db
		port 3306
		database log_db
		adapter mysql2
		username root
		password Rlatjd095980!
		<table>
			table info_login_log
			column_mapping 'LogLevel:level, Timestamp:timestamp, api:api, uid:uid, result:result, playerId:player_id'
		</table>
	</store>

	<store>
		@type sql
		host db
		port 3306
		database log_db
		adapter mysql2
		username root
		password Rlatjd095980!
		<table>
			table info_api_log
			column_mapping 'LogLevel:level, Timestamp:timestamp, api:api, uid:uid, result:result'
		</table>
	</store>

</match>
```
=> login.** �±׸� ���� �α׸� info_login_log ���̺�� info_api_log ���̺� �����Ѵ�.
```conf
``` conf
<match game.**>
	@type copy

	<store>
		@type sql
		host db
		port 3306
		database log_db
		adapter mysql2
		username root
		password Rlatjd095980!
		<table>
			table info_game_log
			column_mapping 'LogLevel:level, Timestamp:timestamp, api:api, uid:uid, result:result, gameKey:game_key, score:score'
		</table>
	</store>

	<store>
		@type sql
		host db
		port 3306
		database log_db
		adapter mysql2
		username root
		password Rlatjd095980!
		<table>
			table info_api_log
			column_mapping 'LogLevel:level, Timestamp:timestamp, api:api, uid:uid, result:result'
		</table>
	</store>

</match>
```
=> game.** �±׸� ���� �α׸� info_game_log ���̺�� info_api_log ���̺� �����Ѵ�.

---
## docker-compose.yml

```yml
---
version: "1.0"

services:
  db:
    image: mydbimage
    container_name: mydb
    build:
      context: ./DB
      dockerfile: Dockerfile
    restart:
      always
    ports:
      - "3306:3306"

  hive:
    image : hiveimage
    container_name: hive
    build:
      context: ./FakeHiveServer/aspnetapp
      dockerfile: Dockerfile
    restart:
      always
    ports:
      - "11501:11501"
    depends_on:
      - db
      - redis

  server:
    image : serverimage
    container_name: server
    build:
      context: ./APIServer/aspnetapp
      dockerfile: Dockerfile
    restart:
      always
    ports:
      - "11500:11500"
    depends_on:
      - db
    volumes:
      - log-volume:/App/log
  
  redis:
    image: redis
    container_name: redis
    restart:
      always
    ports: 
      - "6379:6379"

  fluentd:
    image: fluentd
    container_name: fluentd
    build:
      context: ./fluentd
      dockerfile: Dockerfile
    ports:
      - "24224:24224"
    volumes:
      - log-volume:/App/log

volumes:
  log-volume:    
```

=> https://github.com/thatslifebro/Docker ���� fluentd�� �߰��Ͽ���.

volume�� ���� fluentd �����̳ʿ� server �����̳ʰ� /App/log/ ��θ� �����ϵ��� �Ͽ���.