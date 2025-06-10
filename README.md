# IFMO_DistributedComputing_for_DevOps
Distributed Computing course for DevOps 2025


Запустите плейбук, передавая все необходимые секреты через --extra-vars:

Bash

ansible-playbook -i ansible/inventory.ini ansible/setup.yml \
--extra-vars "ansible_pass=ВАШ_ANSIBLE_ADMIN_ПАРОЛЬ" \
--extra-vars "mysql_root_password=ВАШ_MYSQL_ROOT_ПАРОЛЬ" \
--extra-vars "wp_db_user=ВАШ_WP_DB_ПОЛЬЗОВАТЕЛЬ" \
--extra-vars "wp_db_password=ВАШ_WP_DB_ПАРОЛЬ" \
--extra-vars "wp_db_name=ВАШ_WP_DB_ИМЯ" \
--extra-vars "repl_user=ВАШ_REPL_ПОЛЬЗОВАТЕЛЬ" \
--extra-vars "replication_password=ВАШ_REPL_ПАРОЛЬ"

или внесите значение переменных в example_inventory.ini и укажите его в команде запуска.