# 1.2 — Linux para Engenharia DevOps

## Laboratório 3 — Processos, Serviços, systemd, Logs e Troubleshooting no Linux

Este material complementa a aula teórica **Linux para Engenharia DevOps — Parte 3**.

O objetivo desta prática é compreender processos, serviços, systemd e logs, aplicando esses conceitos em um cenário de investigação e troubleshooting no Linux.

> Os exemplos foram pensados para distribuições Linux com systemd, como Ubuntu, Debian, Fedora e Rocky Linux.

> Execute esta prática em uma VM ou ambiente descartável.

## Pré-requisitos e validação do ambiente

Este laboratório foi preparado para uma distribuição Linux que utilize **systemd como init**, como Ubuntu Server, Debian, Fedora, Rocky Linux ou distribuições equivalentes.

> **Importante:** possuir o comando `systemctl` instalado não significa que o systemd esteja executando como gerenciador do sistema. Containers e alguns ambientes de sandbox costumam usar outro processo como PID 1.

Verifique o processo de PID 1:

```bash
ps -p 1 -o pid,comm,args
```

Em um ambiente adequado ao laboratório, você deverá observar `systemd` como PID 1 ou um caminho equivalente a `/sbin/init` apontando para systemd.

Também verifique:

```bash
systemctl is-system-running
```

Resultados como `running` ou, em alguns ambientes de laboratório, `degraded`, indicam que o systemd está operacional. Se aparecer uma mensagem semelhante a:

```text
System has not been booted with systemd as init system (PID 1)
```

execute a parte de processos normalmente, mas realize as etapas de systemd em uma VM ou host Linux com systemd ativo.

Verifique as ferramentas utilizadas no laboratório:

```bash
command -v ps
command -v pstree
command -v pgrep
command -v top
command -v systemctl
command -v journalctl
command -v systemd-analyze
command -v python3
command -v curl
```

Em Ubuntu/Debian, caso `pstree`, `curl` ou `python3` não estejam disponíveis:

```bash
sudo apt update
sudo apt install -y psmisc curl python3
```

Em Fedora/RHEL/Rocky Linux:

```bash
sudo dnf install -y psmisc curl python3
```

---

## Objetivos

Ao final deste laboratório, você deverá ser capaz de:

- identificar processos e interpretar PID, PPID, usuário, estado, CPU e memória;
- visualizar relações entre processos pai e filho;
- controlar processos utilizando signals;
- investigar informações de processos através do `/proc`;
- compreender a relação entre processo, daemon e serviço;
- inspecionar serviços e units gerenciados pelo systemd;
- diferenciar `active` de `enabled`;
- compreender o papel de targets e dependências;
- consultar e filtrar logs com `journalctl`;
- criar e validar uma unit `.service` própria;
- diagnosticar uma falha utilizando estado, processos e logs;
- validar a funcionalidade de uma aplicação, e não apenas a existência de seu processo.

# 1. Explorando os processos do sistema

Comece observando os processos associados ao terminal atual:

```bash
ps
```

Agora exiba uma visão mais ampla:

```bash
ps -ef
```

Observe principalmente:

```text
UID
PID
PPID
CMD
```

O `PID` identifica o processo e o `PPID` identifica seu processo pai.

Descubra o PID do shell atual:

```bash
echo $$
```

Consulte esse processo:

```bash
ps -p $$ -f
```

Agora exiba apenas algumas propriedades:

```bash
ps -o pid,ppid,user,state,cmd -p $$
```

## Questões

1. Qual é o PID do seu shell?
2. Qual é o PPID dele?
3. Qual usuário está executando o shell?
4. Qual processo aparece como pai do seu shell?

---

# 2. Visualizando a hierarquia de processos

Processos Linux formam uma hierarquia de processos pais e filhos.

Execute:

```bash
pstree
```

Agora inclua os PIDs:

```bash
pstree -p
```

Em uma sessão SSH, por exemplo, uma parte da árvore pode lembrar:

```text
systemd(1)
 └─sshd(...)
    └─sshd(...)
       └─bash(...)
```

A árvore real dependerá do ambiente utilizado.

Agora encontre seu shell na árvore e compare seu PID com o valor retornado por:

```bash
echo $$
```

---

# 3. Criando um processo controlado

Crie um processo simples em background:

```bash
sleep 300 &
```

O shell exibirá algo parecido com:

```text
[1] 1842
```

O número do PID também pode ser obtido imediatamente através de `$!`, que representa o PID do último processo colocado em background:

```bash
SLEEP_PID=$!
echo $SLEEP_PID
```

Consulte o processo:

```bash
ps -p "$SLEEP_PID" -f
```

Exiba informações selecionadas:

```bash
ps -o pid,ppid,user,state,%cpu,%mem,cmd -p "$SLEEP_PID"
```

Como `sleep` passa a maior parte do tempo aguardando, normalmente seu estado aparecerá como sleeping e seu consumo de CPU será muito baixo.

---

# 4. Foreground, background e jobs do shell

Liste os processos em background associados ao shell atual:

```bash
jobs
```

Crie outro processo:

```bash
sleep 500 &
```

Execute novamente:

```bash
jobs
```

Use o número do job apresentado pelo shell para trazê-lo ao foreground. Por exemplo, se ele for `%2`:

```bash
fg %2
```

Suspenda temporariamente o processo com:

```text
Ctrl+Z
```

Verifique:

```bash
jobs
```

Retome o job em background usando o número mostrado pelo shell:

```bash
bg %2
```

> `jobs`, `fg` e `bg` pertencem ao controle de jobs do shell. PID, PPID e signals fazem parte do modelo de processos do sistema operacional.

---

# 5. Comparando um processo em espera com um processo consumindo CPU

Primeiro observe o processo `sleep`:

```bash
ps -o pid,state,%cpu,%mem,cmd -p "$SLEEP_PID"
```

Agora crie propositalmente um processo que consuma CPU:

```bash
yes > /dev/null &
CPU_PID=$!
```

> **Atenção:** `yes` pode utilizar praticamente um núcleo de CPU inteiro. Não deixe esse processo executando além do necessário para o exercício.

Observe-o:

```bash
ps -o pid,state,%cpu,%mem,cmd -p "$CPU_PID"
```

Abra também:

```bash
top
```

Localize o processo `yes` e observe seu `%CPU`.

Saia do `top` pressionando:

```text
q
```

Finalize imediatamente o processo de teste:

```bash
kill "$CPU_PID"
```

Confirme:

```bash
ps -p "$CPU_PID"
```

Se não houver mais saída para o processo, ele foi finalizado.

## Reflexão

Um processo usando 100% de um núcleo de CPU está necessariamente com problema?

Não. Compilação, compressão, processamento de dados e outras tarefas podem legitimamente utilizar toda a CPU disponível. O consumo precisa ser comparado com o comportamento esperado da aplicação.

---

# 6. Observando prioridade com nice

Crie novamente um processo intensivo, agora com um valor de nice maior:

```bash
nice -n 10 yes > /dev/null &
NICE_PID=$!
```

Observe a coluna `NI`:

```bash
ps -o pid,ni,pri,state,%cpu,cmd -p "$NICE_PID"
```

Aumente ainda mais o valor de nice:

```bash
renice 15 -p "$NICE_PID"
```

Consulte novamente:

```bash
ps -o pid,ni,pri,state,%cpu,cmd -p "$NICE_PID"
```

Finalize o processo:

```bash
kill "$NICE_PID"
```

> Um valor de nice maior reduz a prioridade relativa de escalonamento de processos normais. Isso não deve ser tratado como solução genérica para problemas de desempenho.

---

# 7. Investigando um processo através do `/proc`

Crie um novo processo:

```bash
sleep 500 &
PROC_PID=$!
```

Verifique:

```bash
echo $PROC_PID
```

Cada processo possui um diretório correspondente em `/proc`:

```bash
ls /proc/$PROC_PID
```

## 7.1 Status

```bash
cat /proc/$PROC_PID/status
```

Procure campos como:

```text
Name
State
Pid
PPid
Uid
Gid
VmSize
VmRSS
```

Uma forma de filtrar apenas alguns deles é:

```bash
grep -E '^(Name|State|Pid|PPid|Uid|Gid|VmSize|VmRSS):' /proc/$PROC_PID/status
```

## 7.2 Linha de comando

```bash
tr '\0' ' ' < /proc/$PROC_PID/cmdline
echo
```

## 7.3 Executável

```bash
readlink /proc/$PROC_PID/exe
```

## 7.4 Diretório de trabalho

```bash
readlink /proc/$PROC_PID/cwd
```

## 7.5 File descriptors

```bash
ls -l /proc/$PROC_PID/fd
```

É comum encontrar os descritores:

```text
0
1
2
```

associados a `stdin`, `stdout` e `stderr`.

Finalize o processo:

```bash
kill "$PROC_PID"
```

---

# 8. Enviando signals para processos

O comando `kill` é, na prática, uma ferramenta para envio de signals.

Liste os signals disponíveis:

```bash
kill -l
```

Crie um processo:

```bash
sleep 500 &
SIGNAL_PID=$!
```

Envie `SIGTERM`:

```bash
kill -TERM "$SIGNAL_PID"
```

Verifique se o processo continua existindo:

```bash
ps -p "$SIGNAL_PID"
```

Agora crie outro:

```bash
sleep 500 &
SIGNAL_PID=$!
```

Force seu encerramento utilizando `SIGKILL`:

```bash
kill -KILL "$SIGNAL_PID"
```

A forma numérica equivalente é:

```bash
kill -9 "$SIGNAL_PID"
```

A diferença conceitual é importante:

```text
SIGTERM
   ↓
processo recebe o signal
   ↓
pode executar rotinas de encerramento
   ↓
finalização controlada

SIGKILL
   ↓
kernel encerra o processo
   ↓
o processo não pode tratar o signal
```

Em operação, dê preferência a um encerramento controlado antes de recorrer ao `SIGKILL`.

---

# 9. Explorando o systemd

A partir desta seção, confirme novamente que o systemd está operacional:

```bash
ps -p 1 -o pid,comm,args
```

Liste serviços carregados:

```bash
systemctl list-units --type=service
```

Para observar apenas serviços em execução:

```bash
systemctl list-units --type=service --state=running
```

Liste os arquivos de unit disponíveis:

```bash
systemctl list-unit-files --type=service
```

Observe que os dois comandos respondem perguntas diferentes:

- `list-units` trabalha com units carregadas pelo manager;
- `list-unit-files` mostra arquivos de unit encontrados no sistema e seu estado de habilitação.

---

# 10. Inspecionando um serviço existente

`systemd-journald` faz parte do próprio ecossistema systemd e é uma boa unit para inspeção.

Execute:

```bash
systemctl status systemd-journald.service
```

Observe informações como:

```text
Loaded
Active
Main PID
Tasks
Memory
CGroup
```

Obtenha somente o PID principal:

```bash
systemctl show systemd-journald.service -p MainPID --value
```

Armazene-o:

```bash
JOURNALD_PID=$(systemctl show systemd-journald.service -p MainPID --value)
```

Consulte o processo diretamente:

```bash
ps -p "$JOURNALD_PID" -f
```

Temos duas perspectivas sobre o mesmo componente:

```text
systemd
   ↓
service unit
   ↓
processo Linux
```

---

# 11. `active`, `enabled` e `static`

Verifique se o journald está ativo:

```bash
systemctl is-active systemd-journald.service
```

Agora verifique sua configuração de habilitação:

```bash
systemctl is-enabled systemd-journald.service
```

Em muitas distribuições, `systemd-journald.service` aparecerá como:

```text
static
```

Isso é normal.

- `active` descreve o estado atual da unit;
- `enabled` indica que foram criadas associações para iniciar a unit por meio de sua seção `[Install]`;
- `disabled` indica que essas associações não estão habilitadas;
- `static` normalmente indica uma unit que não é habilitada diretamente dessa forma, mas pode ser iniciada por dependências ou outros mecanismos.

Mais adiante criaremos nossa própria unit para observar diretamente a diferença entre `active` e `enabled`.

---

# 12. Inspecionando uma unit file

Exiba a definição utilizada pelo systemd:

```bash
systemctl cat systemd-journald.service
```

Agora consulte propriedades interpretadas pelo manager:

```bash
systemctl show systemd-journald.service \
  -p ActiveState \
  -p SubState \
  -p MainPID \
  -p FragmentPath
```

`systemctl cat` mostra a configuração textual da unit e eventuais drop-ins.

`systemctl show` mostra propriedades que o systemd está utilizando internamente para aquela unit.

---

# 13. Targets e dependências

Descubra o target padrão do sistema:

```bash
systemctl get-default
```

Em servidores, um valor comum é:

```text
multi-user.target
```

Em máquinas com ambiente gráfico, pode aparecer:

```text
graphical.target
```

Explore as dependências de `multi-user.target`:

```bash
systemctl list-dependencies multi-user.target
```

Agora examine as dependências do journald:

```bash
systemctl list-dependencies systemd-journald.service
```

Lembre-se de que **dependência** e **ordem de inicialização** não são exatamente a mesma coisa. Diretivas como `Requires=` e `Wants=` expressam relacionamentos; diretivas como `Before=` e `After=` tratam de ordenação.

---

# 14. Consultando o journal

Visualize eventos recentes:

```bash
journalctl -n 20
```

Exiba apenas eventos do boot atual:

```bash
journalctl -b
```

Para sair do pager, pressione:

```text
q
```

Consulte apenas eventos do journald:

```bash
journalctl -u systemd-journald.service -n 20
```

Consulte eventos recentes por período:

```bash
journalctl --since "10 minutes ago"
```

Combine filtros:

```bash
journalctl -u systemd-journald.service --since "1 hour ago"
```

---

# 15. Filtrando logs por prioridade

Visualize erros ou eventos mais graves:

```bash
journalctl -p err
```

Visualize warnings ou eventos mais graves:

```bash
journalctl -p warning
```

Algumas prioridades utilizadas pelo journal são:

```text
emerg
alert
crit
err
warning
notice
info
debug
```

> Encontrar uma mensagem de erro não prova, por si só, que ela seja a causa do incidente atual. Sempre correlacione componente, horário, sequência de eventos e sintoma observado.

---

# 16. Criando um usuário para o serviço do laboratório

Criaremos uma aplicação simples executada por um usuário de sistema próprio.

Primeiro verifique se o usuário já existe:

```bash
getent passwd labsvc
```

Se não houver saída, crie-o:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  labsvc
```

Confirme:

```bash
getent passwd labsvc
```

E verifique o grupo criado:

```bash
getent group labsvc
```

---

# 17. Criando a aplicação `lab-heartbeat`

Crie o script:

```bash
sudo nano /usr/local/bin/lab-heartbeat.sh
```

Adicione:

```bash
#!/bin/bash

trap 'echo "Recebido SIGTERM. Encerrando corretamente..."; exit 0' TERM
trap 'echo "Recebido SIGINT. Encerrando corretamente..."; exit 0' INT

echo "Aplicação iniciada. PID=$$"

while true
do
    echo "heartbeat pid=$$ timestamp=$(date --iso-8601=seconds)"
    sleep 5
done
```

Torne-o executável:

```bash
sudo chmod 755 /usr/local/bin/lab-heartbeat.sh
```

Confirme as permissões:

```bash
ls -l /usr/local/bin/lab-heartbeat.sh
```

Teste manualmente:

```bash
/usr/local/bin/lab-heartbeat.sh
```

A saída deverá se parecer com:

```text
Aplicação iniciada. PID=...
heartbeat pid=... timestamp=...
heartbeat pid=... timestamp=...
```

Finalize com:

```text
Ctrl+C
```

O `trap` deverá registrar o encerramento antes do processo terminar.

---

# 18. Criando a primeira unit `.service`

Crie:

```bash
sudo nano /etc/systemd/system/lab-heartbeat.service
```

Adicione:

```ini
[Unit]
Description=Laboratório - Heartbeat Service

[Service]
Type=simple
User=labsvc
Group=labsvc
ExecStart=/usr/local/bin/lab-heartbeat.sh

[Install]
WantedBy=multi-user.target
```

Antes de carregar a unit, valide sua sintaxe:

```bash
sudo systemd-analyze verify /etc/systemd/system/lab-heartbeat.service
```

Se não houver erros, recarregue a configuração do manager:

```bash
sudo systemctl daemon-reload
```

---

# 19. Iniciando o serviço

Inicie:

```bash
sudo systemctl start lab-heartbeat.service
```

Verifique:

```bash
systemctl status lab-heartbeat.service
```

Consulte somente o estado:

```bash
systemctl is-active lab-heartbeat.service
```

Resultado esperado:

```text
active
```

Descubra o PID principal:

```bash
HEARTBEAT_PID=$(systemctl show lab-heartbeat.service -p MainPID --value)
echo $HEARTBEAT_PID
```

Investigue o processo diretamente:

```bash
ps -o pid,ppid,user,state,%cpu,%mem,cmd -p "$HEARTBEAT_PID"
```

Consulte também `/proc`:

```bash
grep -E '^(Name|State|Pid|PPid|Uid|Gid):' /proc/$HEARTBEAT_PID/status
```

Agora temos a relação completa:

```text
systemd
  ↓
lab-heartbeat.service
  ↓
lab-heartbeat.sh
  ↓
processo Linux
```

---

# 20. `start` não é `enable`

O serviço está executando. Verifique se está habilitado para inicialização automática:

```bash
systemctl is-enabled lab-heartbeat.service
```

Como executamos apenas `start`, o resultado esperado neste momento é:

```text
disabled
```

Temos, portanto:

```text
active + disabled
```

Agora habilite a unit:

```bash
sudo systemctl enable lab-heartbeat.service
```

Verifique novamente:

```bash
systemctl is-enabled lab-heartbeat.service
```

Resultado esperado:

```text
enabled
```

Observe o link criado:

```bash
ls -l /etc/systemd/system/multi-user.target.wants/lab-heartbeat.service
```

Ele deverá apontar para a unit instalada em `/etc/systemd/system`.

Isso evidencia que:

```text
start  → altera a execução atual
enable → configura participação na inicialização apropriada
```

---

# 21. Consultando os logs do serviço

Nosso script escreve em `stdout`. Como foi iniciado pelo systemd, essa saída é coletada pelo journal na configuração padrão das distribuições systemd.

Visualize os logs:

```bash
journalctl -u lab-heartbeat.service -n 20
```

Acompanhe em tempo real:

```bash
journalctl -u lab-heartbeat.service -f
```

Você deverá observar mensagens semelhantes a:

```text
Aplicação iniciada. PID=...
heartbeat pid=... timestamp=...
heartbeat pid=... timestamp=...
```

Interrompa apenas o acompanhamento com:

```text
Ctrl+C
```

Isso encerra o `journalctl`, não o serviço.

---

# 22. Observando o encerramento com SIGTERM

Em um terminal, acompanhe os logs:

```bash
journalctl -u lab-heartbeat.service -f
```

Em outro terminal, pare o serviço:

```bash
sudo systemctl stop lab-heartbeat.service
```

O script deverá registrar algo semelhante a:

```text
Recebido SIGTERM. Encerrando corretamente...
```

Confirme o estado:

```bash
systemctl is-active lab-heartbeat.service
```

Agora inicie novamente:

```bash
sudo systemctl start lab-heartbeat.service
```

Esse exercício relaciona diretamente:

```text
systemctl stop
      ↓
systemd
      ↓
signal para o processo
      ↓
encerramento da aplicação
```

---

# 23. Restart cria uma nova execução

Descubra o PID atual:

```bash
systemctl show lab-heartbeat.service -p MainPID --value
```

Anote o valor.

Reinicie:

```bash
sudo systemctl restart lab-heartbeat.service
```

Consulte novamente:

```bash
systemctl show lab-heartbeat.service -p MainPID --value
```

O PID deverá ser diferente, pois a execução anterior terminou e uma nova instância do programa foi criada.

---

# 24. Troubleshooting: quebrando propositalmente o serviço

Agora vamos introduzir uma falha controlada.

Pare o serviço:

```bash
sudo systemctl stop lab-heartbeat.service
```

Edite a unit:

```bash
sudo nano /etc/systemd/system/lab-heartbeat.service
```

Troque:

```ini
ExecStart=/usr/local/bin/lab-heartbeat.sh
```

por:

```ini
ExecStart=/usr/local/bin/lab-heartbeat-inexistente.sh
```

Valide a unit:

```bash
sudo systemd-analyze verify /etc/systemd/system/lab-heartbeat.service
```

Dependendo da versão do systemd, a ferramenta poderá alertar que o executável não existe. Isso já constitui uma evidência útil.

Recarregue as configurações:

```bash
sudo systemctl daemon-reload
```

Agora tente iniciar:

```bash
sudo systemctl start lab-heartbeat.service
```

O comando deverá falhar.

---

# 25. Investigando antes de corrigir

Imagine que essa falha ocorreu em produção. Não abra vários arquivos e não altere diversas coisas simultaneamente.

Comece pelo sintoma:

> O serviço não inicia.

Verifique o estado:

```bash
systemctl status lab-heartbeat.service
```

Consulte algumas propriedades:

```bash
systemctl show lab-heartbeat.service \
  -p ActiveState \
  -p SubState \
  -p Result \
  -p MainPID
```

Agora consulte os logs:

```bash
journalctl -u lab-heartbeat.service -n 30
```

Procure mensagens relacionadas à execução de `ExecStart`.

O fluxo da investigação é:

```text
Sintoma
   ↓
serviço não inicia

Estado
   ↓
unit não permanece ativa

Logs
   ↓
falha ao executar ExecStart

Hipótese
   ↓
caminho do executável está incorreto
```

---

# 26. Validando a hipótese

Veja a configuração efetiva da unit:

```bash
systemctl cat lab-heartbeat.service
```

Agora verifique se o arquivo existe:

```bash
ls -l /usr/local/bin/lab-heartbeat-inexistente.sh
```

O comando deverá informar que o arquivo não existe.

Temos agora uma hipótese sustentada por evidências.

---

# 27. Corrigindo e validando

Edite a unit novamente:

```bash
sudo nano /etc/systemd/system/lab-heartbeat.service
```

Restaure:

```ini
ExecStart=/usr/local/bin/lab-heartbeat.sh
```

Valide:

```bash
sudo systemd-analyze verify /etc/systemd/system/lab-heartbeat.service
```

Recarregue:

```bash
sudo systemctl daemon-reload
```

Inicie:

```bash
sudo systemctl start lab-heartbeat.service
```

Valide o estado:

```bash
systemctl is-active lab-heartbeat.service
```

Resultado esperado:

```text
active
```

Valide também os logs:

```bash
journalctl -u lab-heartbeat.service -n 20
```

O importante é perceber que não fizemos:

```text
reiniciar a máquina
reinstalar pacotes
alterar diversas configurações ao mesmo tempo
```

Seguimos:

```text
Sintoma
   ↓
Estado
   ↓
Logs
   ↓
Evidência
   ↓
Hipótese
   ↓
Validação da hipótese
   ↓
Correção
   ↓
Validação final
```

---

# 28. Desafio: processo rodando não significa aplicação saudável

Agora criaremos uma aplicação HTTP simples.

Crie o diretório:

```bash
sudo mkdir -p /srv/lab-web
```

Crie um endpoint de saúde representado por um arquivo:

```bash
echo "healthy" | sudo tee /srv/lab-web/health.txt
```

Garanta que o usuário do serviço consiga ler o conteúdo:

```bash
sudo chmod 755 /srv/lab-web
sudo chmod 644 /srv/lab-web/health.txt
```

Antes de continuar, verifique se a porta `18080` já está sendo utilizada:

```bash
ss -ltn | grep ':18080 '
```

Se não houver saída, a porta está livre para o laboratório.

> Caso `ss` não esteja disponível, prossiga e observe os logs caso o serviço não consiga abrir a porta.

---

# 29. Criando o serviço web

Crie:

```bash
sudo nano /etc/systemd/system/lab-web.service
```

Adicione:

```ini
[Unit]
Description=Laboratório - Web Service

[Service]
Type=simple
User=labsvc
Group=labsvc
WorkingDirectory=/srv/lab-web
ExecStart=/usr/bin/python3 -u -m http.server 18080

[Install]
WantedBy=multi-user.target
```

Valide:

```bash
sudo systemd-analyze verify /etc/systemd/system/lab-web.service
```

Recarregue:

```bash
sudo systemctl daemon-reload
```

Inicie:

```bash
sudo systemctl start lab-web.service
```

Verifique:

```bash
systemctl status lab-web.service
```

Teste a aplicação:

```bash
curl http://localhost:18080/health.txt
```

Resultado esperado:

```text
healthy
```

---

# 30. Criando uma falha funcional

Remova o arquivo utilizado como endpoint de saúde:

```bash
sudo rm /srv/lab-web/health.txt
```

Verifique o estado do serviço:

```bash
systemctl is-active lab-web.service
```

Resultado esperado:

```text
active
```

Agora teste a funcionalidade:

```bash
curl -i http://localhost:18080/health.txt
```

Você deverá receber uma resposta `404`.

Temos então:

```text
processo            ✓ executando
service unit        ✓ active
servidor HTTP       ✓ aceitando conexões
recurso esperado    ✗ indisponível
```

Esse é um conceito central para operação de aplicações:

> **Processo vivo não significa aplicação saudável.**

---

# 31. Correlacionando a falha com os logs

Consulte os logs do serviço web:

```bash
journalctl -u lab-web.service -n 20
```

A requisição que retornou `404` deverá aparecer nos registros do servidor HTTP.

Restaure o arquivo:

```bash
echo "healthy" | sudo tee /srv/lab-web/health.txt
sudo chmod 644 /srv/lab-web/health.txt
```

Teste novamente:

```bash
curl http://localhost:18080/health.txt
```

Resultado esperado:

```text
healthy
```

Agora valide simultaneamente o service manager e a funcionalidade:

```bash
systemctl is-active lab-web.service
curl -fsS http://localhost:18080/health.txt
```

---

# 32. Desafio de troubleshooting

Você recebeu o seguinte chamado:

> A aplicação parou de funcionar. O time informou apenas que “o serviço caiu”.

Sem reiniciar a máquina, responda:

1. A unit existe?
2. Ela está carregada?
3. Qual é seu estado atual?
4. Existe um processo principal associado?
5. Qual é o PID?
6. Qual usuário executa o processo?
7. O que os logs mostram?
8. Houve algum evento imediatamente antes da falha?
9. O problema está na unit, no processo, em um recurso do sistema ou na funcionalidade da aplicação?
10. Qual evidência sustenta sua hipótese?
11. Como validar a hipótese sem realizar alterações desnecessárias?
12. Depois da correção, como provar que o serviço realmente voltou a funcionar?

Ferramentas que podem ajudar:

```bash
systemctl status
systemctl show
systemctl cat
systemctl is-active
systemctl is-enabled
journalctl
ps
pstree
pgrep
/proc
curl
```

Não existe obrigação de utilizar todos os comandos. A escolha da ferramenta deve partir da pergunta que você está tentando responder.

---

# 33. Limpando o ambiente

Pare e desabilite os serviços:

```bash
sudo systemctl disable --now lab-heartbeat.service
sudo systemctl disable --now lab-web.service
```

Remova as units:

```bash
sudo rm -f /etc/systemd/system/lab-heartbeat.service
sudo rm -f /etc/systemd/system/lab-web.service
```

Remova os arquivos utilizados:

```bash
sudo rm -f /usr/local/bin/lab-heartbeat.sh
sudo rm -rf /srv/lab-web
```

Remova o usuário de laboratório:

```bash
sudo userdel labsvc
```

Se o grupo permanecer no sistema, remova-o:

```bash
getent group labsvc && sudo groupdel labsvc
```

Recarregue a configuração do systemd:

```bash
sudo systemctl daemon-reload
```

Limpe estados de falha antigos:

```bash
sudo systemctl reset-failed
```

---

# 34. Checklist final

Ao concluir o laboratório, você deve conseguir explicar a diferença entre:

```text
programa × processo

PID × PPID

processo × daemon × serviço

foreground × background

SIGTERM × SIGKILL

start × stop × restart

active × enabled

enabled × static

unit × unit file

dependência × ordem

processo executando × aplicação saudável

status × logs

sintoma × causa
```

E deve compreender o modelo:

```text
Aplicação
    ↓
Processo Linux
    ↓
Service unit
    ↓
systemd
    ↓
Journal
    ↓
Evidências
    ↓
Troubleshooting
```

---

# 35. Resultado esperado

O objetivo deste laboratório não é só aprender comandos como:

```bash
ps
systemctl
journalctl
```

O objetivo é desenvolver um processo de investigação:

```text
O que deveria estar acontecendo?
            ↓
O que está acontecendo de fato?
            ↓
Qual evidência demonstra isso?
            ↓
Qual componente pode explicar a diferença?
            ↓
Como validar essa hipótese?
            ↓
Como corrigir com a menor alteração necessária?
            ↓
Como provar que o serviço voltou a funcionar?
```

Esse modelo continuará válido quando os próximos módulos introduzirem containers, Kubernetes, health checks, observabilidade e workloads distribuídos.

---

# Referências

**NEMETH, E.; SNYDER, G.; HEIN, T. R.; WHALEY, B.; MACKIN, D. UNIX and Linux System Administration Handbook. 5. ed. Pearson, 2018. Caps. 1, 2 e 4.**

**KIM, G.; HUMBLE, J.; DEBOIS, P.; WILLIS, J.; FORSGREN, N. The DevOps Handbook: How to Create World-Class Agility, Reliability, & Security in Technology Organizations. 2. ed. IT Revolution Press, 2021. Parte IV, Caps. 14–15.**
