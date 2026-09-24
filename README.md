# diploma-ansible

Ansible-конфигурация для развёртывания Kubernetes-кластера в Yandex Cloud
через Kubespray v2.31.0.

## Требования

- Python 3.13 (venv)
- Ansible 2.18.x
- Kubespray v2.31.0 в /Users/kovalevski/kubespray
- SSH-ключ ~/.ssh/id_ed25519 (пользователь ubuntu)

## Состав

- inventory.ini.example — шаблон inventory
- ansible.cfg — конфиг Ansible с корректными путями Kubespray,
  host_key_checking = True, pipelining = False
  (необходимо для корректной работы raw-модулей Kubespray под sudo)
- group_vars/all.yml — параметры Kubespray: локальный kubeconfig,
  внешний IP в SAN сертификата API-сервера

## Как использовать

1. Узнать актуальные IP узлов: yc compute instance list

2. Скопировать пример inventory и заполнить реальные IP:
   cp inventory.ini.example inventory.ini

3. Активировать venv Kubespray:
   source /Users/kovalevski/kubespray-venv/bin/activate
   rehash

4. Запустить установку:
   export ANSIBLE_CONFIG=$(pwd)/ansible.cfg
   cd /Users/kovalevski/kubespray
   ansible-playbook -i /Users/kovalevski/diploma-ansible/inventory.ini cluster.yml -b --become-method=sudo

5. Kubeconfig появится в artifacts/admin.conf. Скопировать:
   cp /Users/kovalevski/ansible_yc/artifacts/admin.conf ~/.kube/config
   chmod 600 ~/.kube/config

## Особенности

- Workers preemptible. При остановке ВМ внешние IP меняются, ноды получают
  статус NotReady. Поднять обратно: yc compute instance list;
  yc compute instance start <worker-id>. Внутренние IP остаются
  неизменными, inventory можно не пересобирать до следующего cluster.yml.

- SSH host keys. При смене внешнего IP воркера запись в ~/.ssh/known_hosts
  становится неактуальной. Обновить через ssh-keyscan с control-plane:
  ssh ubuntu@<control-plane-ip> 'ssh-keyscan -t ed25519 <worker-internal-ip>'
  и добавить запись в known_hosts с новым внешним IP.

- Проверка host_key_checking. Убедитесь, что используется именно этот
  ansible.cfg, а не /Users/kovalevski/kubespray/ansible.cfg (в нём
  host_key_checking=False и UserKnownHostsFile=/dev/null).

- pipelining = False включён намеренно. С pipelining = True первый
  raw-модуль Kubespray (bootstrap_os) зависает на
  "Timeout waiting for privilege escalation prompt" при работе через sudo.
