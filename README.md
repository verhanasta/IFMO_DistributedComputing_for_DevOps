# IFMO_DistributedComputing_for_DevOps
Distributed Computing course for DevOps 2025


my-wordpress-project/
│
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions workflow
│
├── ansible/
│   ├── roles/
│   │   ├── wordpress/
│   │   │   ├── tasks/
│   │   │   │   └── main.yml     # Задачи для WordPress
│   │   │   └── templates/
│   │   │       └── wp-config.php.j2  # Шаблон конфигурации WordPress
│   │   ├── mysql/
│   │   │   ├── tasks/
│   │   │   │   └── main.yml     # Задачи для MySQL
│   │   │   └── templates/
│   │   │       └── my.cnf.j2     # Шаблон конфигурации MySQL
│   │   └── mysql_replica/
│   │       ├── tasks/
│   │       │   └── main.yml     # Задачи для реплики MySQL
│   │       └── templates/
│   │           └── my.cnf.j2     # Шаблон конфигурации реплики
│   ├── setup.yml                # Ansible playbook для запуска всех ролей
│   └── docker-compose.yml # Docker Compose файл для WordPress и баз данных
│
└── README.md                    # Описание проекта


# === Файл: ansible/setup.yml ===
# Главный playbook: запускает роли для WordPress и MySQL (master + replica)
- hosts: wordpress
  become: true
  roles:
    - mysql
    - mysql_replica
    - wordpress


# === Файл: ansible/inventory.ini ===
# Инвентори-файл с хостом для удалённого сервера
[wordpress]
your.remote.host ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa


# === Файл: ansible/roles/mysql/tasks/main.yml ===
# Настройка основной базы MySQL (мастера)
- name: Установка MySQL
  apt:
  name: mysql-server
  state: present
  update_cache: true

- name: Копирование my.cnf
  template:
  src: my.cnf.j2
  dest: /etc/mysql/my.cnf
  notify: Перезапуск MySQL

- name: Запуск MySQL и включение автозапуска
  service:
  name: mysql
  state: started
  enabled: true

- name: Создание репликационного пользователя
  mysql_user:
  name: replica
  password: repl_pass
  priv: '*.*:REPLICATION SLAVE'
  host: '%'
  state: present

- name: Получение бинарного лога и позиции
  mysql_replication:
  mode: getprimary
  register: master_status

- name: Сохранение master log info в файл
  copy:
  content: |
  log_file={{ master_status.File }}  # Имя бинарного лога мастера
  log_pos={{ master_status.Position }}  # Позиция в бинарном логе
  dest: /tmp/master_status.txt


# === Файл: ansible/roles/mysql/templates/my.cnf.j2 ===
[mysqld]
bind-address = 0.0.0.0
server-id = 1  # Уникальный идентификатор сервера. Должен отличаться у каждого узла (мастер = 1, реплика = 2 и т.д.)
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = wordpress  # Указывает, какие базы данных реплицируются (только `wordpress` в данном случае)


# === Файл: ansible/roles/mysql_replica/tasks/main.yml ===
# Настройка реплики MySQL
- name: Установка MySQL на реплике
  apt:
  name: mysql-server
  state: present
  update_cache: true

- name: Копирование my.cnf на реплику
  template:
  src: my.cnf.j2
  dest: /etc/mysql/my.cnf
  notify: Перезапуск MySQL

- name: Запуск и включение MySQL
  service:
  name: mysql
  state: started
  enabled: true

- name: Копирование master log info
  fetch:
  src: /tmp/master_status.txt
  dest: /tmp/master_status.txt
  flat: yes

- name: Чтение параметров логов мастера
  set_fact:
  log_file: "{{ lookup('file', '/tmp/master_status.txt') | regex_search('log_file=(.*)', '\\1') }}"
  log_pos: "{{ lookup('file', '/tmp/master_status.txt') | regex_search('log_pos=(.*)', '\\1') }}"

- name: Настройка мастера
  mysql_replication:
  mode: changereplica
  master_host: "{{ hostvars['your.remote.host'].inventory_hostname }}"
  master_user: replica  # Имя пользователя для подключения к мастеру
  master_password: repl_pass  # Пароль пользователя репликации
  master_log_file: "{{ log_file }}"  # Имя бинарного лога мастера
  master_log_pos: "{{ log_pos | int }}"  # Позиция в бинарном логе мастера

- name: Запуск репликации
  mysql_replication:
  mode: startreplica


# === Файл: ansible/roles/mysql_replica/templates/my.cnf.j2 ===
[mysqld]
bind-address = 0.0.0.0
server-id = 2  # Уникальный идентификатор сервера (реплика)
relay_log = /var/log/mysql/mysql-relay-bin.log


# === Файл: ansible/roles/wordpress/tasks/main.yml ===
- name: Установка Docker и docker-compose
  apt:
  name: [ docker.io, docker-compose ]
  state: present
  update_cache: true

- name: Копирование docker-compose файла
  copy:
  src: ../docker-compose.yml  # Файл теперь находится в директории ansible/
  dest: /home/ubuntu/docker-compose.yml

- name: Запуск docker-compose
  command: docker-compose up -d
  args:
  chdir: /home/ubuntu


# === Файл: ansible/roles/wordpress/templates/wp-config.php.j2 ===
<?php
/** Настройки подключения к базе данных **/
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'root' );
define( 'DB_PASSWORD', 'rootpass' );
define( 'DB_HOST', 'db-master:3306' );

/* Остальные настройки */
define( 'DB_CHARSET', 'utf8' );
define( 'DB_COLLATE', '' );

/* Секретные ключи */
define('AUTH_KEY', 'your-secret-key');
// и т.д.

$table_prefix = 'wp_';
define( 'WP_DEBUG', false );

if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}
require_once ABSPATH . 'wp-settings.php';


# === Файл: ansible/docker-compose.yml ===
version: '3.8'

services:
  db-master:
    image: mysql:5.7
    container_name: db-master
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
    volumes:
      - db_master_data:/var/lib/mysql  # Здесь сохраняются данные MySQL-мастера
    ports:
      - "3306:3306"
    networks:
      - wpnet

  db-replica:
    image: mysql:5.7
    container_name: db-replica
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - db_replica_data:/var/lib/mysql  # Здесь сохраняются данные MySQL-реплики
    networks:
      - wpnet
    depends_on:
      - db-master

  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db-master:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: root
      WORDPRESS_DB_PASSWORD: rootpass
    volumes:
      - wordpress_data:/var/www/html  # Здесь сохраняются данные WordPress: плагины, темы, media и т.д.
    networks:
      - wpnet
    depends_on:
      - db-master

volumes:
  db_master_data:    # Именованный том для хранения данных основного MySQL
  db_replica_data:   # Именованный том для хранения данных реплики MySQL
  wordpress_data:    # Именованный том для хранения данных сайта WordPress

networks:
  wpnet:             # Общая сеть для взаимодействия всех контейнеров
