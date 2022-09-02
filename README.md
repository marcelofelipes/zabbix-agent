Neste primeiro bloco faço a instalação do python e python-pip, vão ser úteis na hora de usar o modulo do Ansible;

---
- name: install Python e python-pip
  apt:
    name: ['python', 'python-pip']
    update_cache: yes
    state: presentwhen: (ansible_os_family == "Debian") or (ansible_os_family == "Ubuntu")
Agora eu removo somente o agente zabbix da maquina cliente.

- name: Removendo Zabbix-Agent
  apt: 
    name: zabbix-agent
    state: removed
Neste bloco armazeno as variáveis: SO, SOVERSION, ipack, hname e hip.

- name: Registro de sistema operacional
  shell: lsb_release -si | tr 'A-Z' 'a-z'
  register: SO

- name: Registro da versao do sistema operacional
  shell: lsb_release -sc
  register: SOVERSION
 
- name: Registro do nome da maquina 
  shell: hostname
  register: hname

- name: Registro de IP
  shell: hostname -I | awk '{print $1}'register: hip

- name: Registro de pacote instalado
  shell: dpkg -l | grep zabbix-agent; echo $?
  register: ipack  
  
Neste ponto vejo se o pacote esta instalado utilizando a variável ipack, se estiver instalado eu removo os pacotes, se não estiver instalado pula para a próxima etapa, depois é baixado os pacotes .deb de instalação do agente, neste caso a versão é a 4.0 do agente, em seguida ele irá fazer a instalação dos novos pacotes zabbix-agent e depois atualizar os repositórios.

- name: Removendo bibliotecas Zabbix-Agent
  action: raw dpkg --purge zabbix-release && dpkg --purge zabbix-agent
  when: ipack.stdout != "1"


- name: Baixando biblioteca Zabbix-Agent 4.0action: raw wget https://repo.zabbix.com/zabbix/4.0/"{{ SO.stdout }}"/pool/main/z/zabbix-release_4.0.2%2B"{{ SOVERSION.stdout }}"_all.deb
  
- name: Executando novos repositorios .deb
  action: raw dpkg -i zabbix-release_4.0-2+"{{ SOVERSION.stdout }}"_all.deb


- name: Atualizando repositorios
  action: raw apt-get update
  
Neste passo eu instalo o zabbix-agent e copio o arquivo de configuração já com as informações, somente uma que vou adicionar com a variável de hostname porque cada máquina tem seu próprio hostname, e por último o agente é iniciado (start).

- name: Install agent-zabbix
  apt:
    name: zabbix-agent
    state: installed
       
- name: Copiando arquivo de configuracao do zabbix-agent
  copy:
    src: zabbix/zabbix_agentd.conf
    dest: /etc/zabbix/

- name: adicionado nome do host
  action: raw sed -i "s/Hostname=zabbix/Hostname="{{ hname.stdout }}"/g" /etc/zabbix/zabbix_agentd.conf

- name: Iniciar agente zabbix
  service:
    name: zabbix-agent
    state: started
    enabled: yes
Neste ponto usamos o python-pip para copilar os pacotes zabbix-pip, aqui estou copiando o tar.gz para dentro da maquina cliente e instalando usando o pip install. Pronto já podemos utilizar o modulo do Zabbix no Ansible, lembrando que o zabbix-pip tem que estar instalado no Servidor em que esta o Ansible e no host que vai ser monitorado.

- name: enviando arquivo zabbix pip
  copy: 
    src: zabbix/zabbix-api-0.5.3.tar.gz
    dest: /tmp/    
 
- name: instalando o zabbix-api
  shell: pip install /tmp/zabbix-api-0.5.3.tar.gz
Aqui entra o legal de tudo, usando o modulo do Ansible-Zabbix ele vai bater no front end e já vai incluir todos os dados da máquina. Eu inclui somente o template "Template OS Linux", mas você pode colocar qualquer template, após finalizar esta etapa o Zabbix vai estar recebendo os dados e monitorando o seu host. Não é massa!!! \o/

Eu passo as variáveis hip e hname que vai incluir o ip da maquina e o nome da máquina no monitoramento.

- name: Colocando a maquina no monitoramento
  local_action:
    module: zabbix_host
    server_url: "sua URL zabbix"
    login_user: "Seu usuario"
    login_password: "Sua senha"
    host_name: '{{ hname.stdout }}'
    visible_name: '{{ hname.stdout }}'
    description: Instalacao e configuracao via ansible
    host_groups:
      - Linux servers
    link_templates:
      - Template OS Linux
    status: enabled
    interfaces:
      - type: 1
        main: 1
        useip: 1
        ip: '{{ hip.stdout }}'
        dns: ""
        port: 10050

Agora por último um restart no agente

- name: Restart agente zabbix
  service:
    name: zabbix-agent
    state: restarted
    enabled: yes
