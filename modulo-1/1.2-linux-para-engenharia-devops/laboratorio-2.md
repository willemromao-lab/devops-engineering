# 1.2 — Linux para Engenharia DevOps

## Laboratório 2 — Usuários, Grupos, Permissões, sudo e Gerenciamento de Pacotes

Este material complementa a aula teórica **Linux para Engenharia DevOps — Parte 2**.

O objetivo desta prática é compreender como o Linux representa identidades, controla acesso a arquivos e diretórios, delega privilégios administrativos e gerencia software instalado no sistema.

Ao longo do laboratório vamos simular um cenário semelhante ao encontrado em ambientes reais:

- usuários humanos;
- uma conta utilizada por uma aplicação;
- um grupo responsável por deployment;
- arquivos de configuração;
- arquivos sensíveis;
- diretórios compartilhados;
- privilégios administrativos específicos;
- instalação e inspeção de pacotes.

> Os exemplos foram pensados principalmente para **Ubuntu e Debian**.

> Execute esta prática em uma VM ou ambiente descartável.

## Pré-requisitos e validação do ambiente

Esta prática foi preparada para **Ubuntu/Debian** e pressupõe um usuário comum com acesso administrativo por `sudo`.

Antes de começar, confirme que você **não está em um shell root**:

```bash
id -u
```

O resultado esperado para seu usuário normal é um número diferente de:

```text
0
```

Se o resultado for `0`, saia do shell root e utilize um usuário comum com `sudo`. Vários testes desta prática dependem dessa separação.

Verifique também se as ferramentas principais estão disponíveis:

```bash
command -v sudo adduser useradd groupadd usermod getent visudo apt dpkg
```

Valide seu acesso ao `sudo`:

```bash
sudo -v
```

> As etapas de usuários, grupos e permissões funcionam sem acesso à Internet. A parte de instalação de pacotes precisa de acesso aos repositórios configurados no sistema.

### Verificando resíduos de uma execução anterior

Esta prática utiliza nomes específicos. Antes de iniciar, verifique se eles já existem:

```bash
getent passwd ana-lab
getent passwd bruno-lab
getent passwd payments-svc
getent group payments-deploy
```

Confira também:

```bash
test -e /srv/payments-lab && echo "Encontrado: /srv/payments-lab"
test -e /etc/sudoers.d/devops-lab && echo "Encontrado: /etc/sudoers.d/devops-lab"
```

Se algum desses recursos for de uma execução anterior deste mesmo laboratório, execute a seção **Limpando o laboratório** antes de reiniciar a prática.

---

## Objetivos

Ao final desta prática, você deverá ser capaz de:

- identificar UID, GID e grupos associados a um usuário;
- compreender a relação entre `/etc/passwd`, `/etc/shadow` e `/etc/group`;
- criar usuários humanos e contas de serviço;
- criar e utilizar grupos;
- compreender propriedade de arquivos e diretórios;
- modificar owner e group;
- interpretar permissões `r`, `w` e `x`;
- utilizar permissões simbólicas e numéricas;
- compreender diferenças de permissões entre arquivos e diretórios;
- utilizar `umask`;
- compreender o uso de `setgid` e sticky bit em diretórios;
- delegar uma operação administrativa específica através de `sudo`;
- identificar pacotes instalados;
- compreender a relação entre `apt` e `dpkg`;
- consultar informações e arquivos pertencentes a um pacote;
- atualizar os metadados dos repositórios;
- simular atualizações antes de aplicá-las;
- instalar um pacote;
- aplicar esses conceitos em um pequeno cenário de controle de acesso.

---

# 1. Antes de começar

Descubra qual usuário está executando a sessão:

```bash
whoami
```

Visualize sua identidade completa:

```bash
id
```

Uma saída comum é semelhante a:

```text
uid=1000(aluno) gid=1000(aluno) groups=1000(aluno),27(sudo)
```

Observe três informações importantes:

```text
UID      → identifica o usuário

GID      → identifica o grupo principal

groups   → grupos dos quais o usuário participa
```

O Linux utiliza principalmente os identificadores numéricos para representar usuários e grupos.

---

# 2. Descobrindo os grupos do usuário

Execute:

```bash
groups
```

Ou:

```bash
id
```

Um usuário pode possuir:

```text
grupo principal
+
grupos suplementares
```

Os grupos permitem conceder acesso coletivo a recursos sem compartilhar contas entre pessoas diferentes.

---

# 3. Consultando a base de usuários

Visualize as primeiras entradas conhecidas pelo sistema:

```bash
getent passwd | head
```

`getent` consulta a base de identidades configurada no sistema.

Em uma máquina simples, essas informações normalmente vêm de:

```text
/etc/passwd
```

Porém, em ambientes corporativos elas também podem vir de serviços como LDAP.

---

# 4. O arquivo /etc/passwd

Visualize:

```bash
head /etc/passwd
```

Cada linha representa uma conta.

Um exemplo simplificado:

```text
usuario:x:1000:1000:Nome:/home/usuario:/bin/bash
```

Os campos representam:

```text
login
:
senha-placeholder
:
UID
:
GID principal
:
informações adicionais
:
diretório home
:
shell de login
```

Observe sua própria entrada:

```bash
getent passwd "$(whoami)"
```

---

# 5. O arquivo /etc/shadow

As informações de autenticação sensíveis não ficam diretamente em `/etc/passwd`.

Tente:

```bash
cat /etc/shadow
```

Um usuário comum deverá receber algo semelhante a:

```text
Permission denied
```

Em vez de exibir o conteúdo como administrador, observe as permissões do arquivo:

```bash
ls -l /etc/shadow
```

E seus metadados:

```bash
sudo stat /etc/shadow
```

O acesso é restrito porque esse arquivo contém informações sensíveis relacionadas às senhas das contas.

> Senhas não são armazenadas em texto puro nesse arquivo. Nesta prática não exibiremos os hashes armazenados em `/etc/shadow`.

---

# 6. O arquivo /etc/group

Visualize algumas entradas:

```bash
head /etc/group
```

Também podemos consultar através de:

```bash
getent group | head
```

Uma entrada possui estrutura semelhante a:

```text
grupo:x:GID:membros
```

Por exemplo:

```text
developers:x:1005:ana,carlos
```

---

# 7. Preparando os usuários do laboratório

Vamos criar duas contas que representarão engenheiros.

Primeiro verifique se os nomes já existem:

```bash
getent passwd ana-lab
getent passwd bruno-lab
```

Se não houver saída, podemos criá-los.

Execute:

```bash
sudo adduser --disabled-password --gecos "" ana-lab
```

Depois:

```bash
sudo adduser --disabled-password --gecos "" bruno-lab
```

A opção utilizada evita a necessidade de definir senhas, pois vamos acessar essas identidades através do usuário administrativo do laboratório.

---

# 8. Inspecionando os novos usuários

Execute:

```bash
id ana-lab
```

Depois:

```bash
id bruno-lab
```

Observe que cada usuário possui:

```text
UID
GID
grupos
```

Consulte também:

```bash
getent passwd ana-lab
```

E:

```bash
getent passwd bruno-lab
```

---

# 9. Diretórios home

Confira:

```bash
ls -ld /home/ana-lab
```

E:

```bash
ls -ld /home/bruno-lab
```

Os diretórios pessoais normalmente pertencem ao próprio usuário.

Verifique:

```bash
stat /home/ana-lab
```

Observe principalmente:

```text
Uid
Gid
Access
```

---

# 10. Criando uma conta de serviço

Aplicações não precisam necessariamente executar utilizando uma conta humana.

Vamos criar uma identidade específica para uma aplicação:

```bash
sudo useradd \
  --system \
  --user-group \
  --no-create-home \
  --shell /usr/sbin/nologin \
  payments-svc
```

Consulte:

```bash
getent passwd payments-svc
```

Observe principalmente o shell:

```text
/usr/sbin/nologin
```

Essa conta foi criada para executar processos, não para uma pessoa abrir uma sessão interativa.

---

# 11. Usuário humano × usuário de serviço

Temos agora:

```text
ana-lab
    ↓
usuário humano

bruno-lab
    ↓
usuário humano

payments-svc
    ↓
identidade da aplicação
```

Tente executar um comando utilizando a identidade do serviço:

```bash
sudo -u payments-svc id
```

Mesmo sem possuir login interativo, essa identidade pode executar processos.

Esse é um padrão extremamente comum em servidores Linux.

---

# 12. Criando um grupo de deployment

Agora criaremos um grupo que representa pessoas autorizadas a trabalhar nos arquivos da aplicação.

```bash
sudo groupadd payments-deploy
```

Confira:

```bash
getent group payments-deploy
```

---

# 13. Adicionando usuários ao grupo

Adicione Ana:

```bash
sudo usermod -aG payments-deploy ana-lab
```

Depois Bruno:

```bash
sudo usermod -aG payments-deploy bruno-lab
```

Confira:

```bash
id ana-lab
```

E:

```bash
id bruno-lab
```

Você deverá encontrar:

```text
payments-deploy
```

entre os grupos suplementares.

> A opção `-aG` é importante. `-G` sem `-a` pode substituir a lista de grupos suplementares do usuário.

---

# 14. Consultando o grupo

Execute:

```bash
getent group payments-deploy
```

Uma saída semelhante deverá aparecer:

```text
payments-deploy:x:...:ana-lab,bruno-lab
```

Temos agora:

```text
             ┌───────────────┐
ana-lab ────▶│               │
             │ payments-     │
             │ deploy        │
bruno-lab ──▶│               │
             └───────────────┘
```

---

# 15. Preparando a aplicação do laboratório

Vamos criar uma estrutura semelhante a uma pequena aplicação instalada em um servidor.

```bash
sudo mkdir -p /srv/payments-lab/releases
sudo mkdir -p /srv/payments-lab/config
sudo mkdir -p /srv/payments-lab/logs
```

Confira:

```bash
find /srv/payments-lab
```

---

# 16. Criando arquivos da aplicação

Crie um arquivo de configuração:

```bash
sudo tee /srv/payments-lab/config/app.env > /dev/null <<'EOF'
APP_NAME=payments-api
APP_ENV=production
PORT=8080
LOG_LEVEL=INFO
EOF
```

Crie um arquivo sensível:

```bash
sudo tee /srv/payments-lab/config/secret.env > /dev/null <<'EOF'
DATABASE_PASSWORD=lab-password
EOF
```

> A senha é fictícia e existe apenas para o laboratório.

Crie também um log:

```bash
sudo tee /srv/payments-lab/logs/application.log > /dev/null <<'EOF'
INFO application started
INFO connected to database
EOF
```

---

# 17. Observando a propriedade inicial

Execute:

```bash
ls -l /srv/payments-lab/config
```

Como os arquivos foram criados através de `sudo`, provavelmente pertencem a:

```text
root
```

Isso não representa corretamente nosso cenário.

A aplicação deverá possuir seus próprios arquivos.

---

# 18. Alterando o proprietário com chown

Execute:

```bash
sudo chown payments-svc:payments-deploy /srv/payments-lab/config/app.env
```

Confira:

```bash
ls -l /srv/payments-lab/config/app.env
```

A estrutura é:

```text
owner:group
```

Neste caso:

```text
payments-svc:payments-deploy
```

Ou seja:

```text
payments-svc      → proprietário

payments-deploy   → grupo proprietário
```

---

# 19. Alterando propriedade recursivamente

A aplicação também deverá possuir o diretório de logs.

Execute:

```bash
sudo chown -R payments-svc:payments-svc /srv/payments-lab/logs
```

Confira:

```bash
ls -ld /srv/payments-lab/logs
```

Depois:

```bash
ls -l /srv/payments-lab/logs
```

A opção:

```text
-R
```

faz a operação ocorrer recursivamente.

Use operações recursivas com cuidado em ambientes reais.

---

# 20. Entendendo ls -l

Execute:

```bash
ls -l /srv/payments-lab/config/app.env
```

Uma saída pode ser semelhante a:

```text
-rw-r--r-- 1 payments-svc payments-deploy ... app.env
```

Podemos separar:

```text
-  rw-  r--  r--
│   │    │    │
│   │    │    └── others
│   │    └─────── group
│   └──────────── owner
└──────────────── tipo
```

---

# 21. As três classes de permissão

As permissões tradicionais são divididas em:

```text
owner
group
others
```

Cada grupo possui três posições:

```text
r   read

w   write

x   execute
```

Por exemplo:

```text
rw-r-----
```

significa:

```text
owner   → leitura e escrita

group   → leitura

others  → nenhum acesso
```

---

# 22. Alterando permissões simbolicamente

Defina o arquivo de configuração para que:

```text
owner  → leia e escreva

group  → apenas leia

others → nenhum acesso
```

Execute:

```bash
sudo chmod u=rw,g=r,o= /srv/payments-lab/config/app.env
```

Confira:

```bash
ls -l /srv/payments-lab/config/app.env
```

Resultado esperado:

```text
-rw-r-----
```

---

# 23. Testando acesso como outro usuário

Antes precisamos garantir que o grupo consiga atravessar a estrutura de diretórios.

Execute:

```bash
sudo chown payments-svc:payments-deploy /srv/payments-lab
sudo chown payments-svc:payments-deploy /srv/payments-lab/config
```

Depois:

```bash
sudo chmod 750 /srv/payments-lab
sudo chmod 750 /srv/payments-lab/config
```

Agora teste como Ana:

```bash
sudo -u ana-lab cat /srv/payments-lab/config/app.env
```

Ana pertence ao grupo:

```text
payments-deploy
```

e deverá conseguir ler o arquivo.

---

# 24. Tentando modificar o arquivo

Execute:

```bash
sudo -u ana-lab sh -c \
  'echo "DEBUG=true" >> /srv/payments-lab/config/app.env'
```

A operação deverá ser recusada.

Por quê?

O arquivo possui:

```text
owner   rw-

group   r--

others  ---
```

Ana pertence ao grupo proprietário, mas esse grupo não possui:

```text
w
```

---

# 25. Testando outro usuário do grupo

Execute:

```bash
sudo -u bruno-lab cat /srv/payments-lab/config/app.env
```

Bruno também deverá conseguir ler o arquivo.

Tente escrever:

```bash
sudo -u bruno-lab sh -c \
  'echo "TEST=true" >> /srv/payments-lab/config/app.env'
```

Novamente, a operação deverá ser negada.

---

# 26. Protegendo um segredo

Agora configure o arquivo sensível.

```bash
sudo chown payments-svc:payments-svc \
  /srv/payments-lab/config/secret.env
```

Depois:

```bash
sudo chmod 600 /srv/payments-lab/config/secret.env
```

Confira:

```bash
ls -l /srv/payments-lab/config/secret.env
```

Resultado:

```text
-rw-------
```

Apenas o proprietário possui acesso.

---

# 27. Testando o segredo como aplicação

Execute:

```bash
sudo -u payments-svc \
  cat /srv/payments-lab/config/secret.env
```

A leitura deverá funcionar.

---

# 28. Testando o segredo como desenvolvedor

Agora:

```bash
sudo -u ana-lab \
  cat /srv/payments-lab/config/secret.env
```

Resultado esperado:

```text
Permission denied
```

Mesmo sendo integrante do grupo de deployment, Ana não recebeu acesso ao segredo.

Isso demonstra uma ideia importante:

> Participar de um projeto não significa necessariamente possuir acesso a todos os recursos daquele projeto.

---

# 29. Permissões numéricas

As permissões também podem ser representadas numericamente.

Os valores básicos são:

```text
r = 4

w = 2

x = 1
```

As permissões são combinadas.

Por exemplo:

```text
7 = 4 + 2 + 1 = rwx

6 = 4 + 2     = rw-

5 = 4 + 1     = r-x

4 = 4         = r--
```

---

# 30. Interpretando 750

Considere:

```text
750
```

Podemos separar:

```text
7      5      0
│      │      │
owner  group  others
```

Portanto:

```text
owner   → rwx

group   → r-x

others  → ---
```

Configure:

```bash
sudo chmod 750 /srv/payments-lab/config
```

Confira:

```bash
ls -ld /srv/payments-lab/config
```

---

# 31. O significado de x em diretórios

Em arquivos:

```text
x → executar
```

Em diretórios, a interpretação é diferente.

Em um diretório:

```text
r → listar nomes

w → criar/remover/renomear entradas

x → atravessar o diretório
```

Vamos experimentar isso de forma controlada.

---

# 32. Criando um diretório de teste

Execute:

```bash
sudo mkdir /srv/payments-lab/permission-test
```

Configure:

```bash
sudo chown root:payments-deploy \
  /srv/payments-lab/permission-test
```

Depois:

```bash
sudo chmod 740 /srv/payments-lab/permission-test
```

O grupo possui:

```text
r--
```

mas não possui:

```text
x
```

---

# 33. Listar não significa necessariamente acessar

Crie um arquivo e defina explicitamente suas permissões para que o teste não dependa da `umask` do usuário administrativo:

```bash
sudo touch /srv/payments-lab/permission-test/file.txt
sudo chmod 644 /srv/payments-lab/permission-test/file.txt
```

Agora tente:

```bash
sudo -u ana-lab \
  ls /srv/payments-lab/permission-test
```

Você poderá visualizar os nomes das entradas, mas isso não significa que consiga acessar seu conteúdo.

Tente:

```bash
sudo -u ana-lab \
  cat /srv/payments-lab/permission-test/file.txt
```

O acesso deverá falhar porque Ana não possui `x` no diretório.

Adicione `x` ao grupo:

```bash
sudo chmod 750 /srv/payments-lab/permission-test
```

Teste novamente.

---

# 34. Criando uma área compartilhada

Agora vamos preparar o diretório onde membros do grupo poderão publicar releases.

```bash
sudo chown payments-svc:payments-deploy \
  /srv/payments-lab/releases
```

Configure:

```bash
sudo chmod 2770 /srv/payments-lab/releases
```

Confira:

```bash
ls -ld /srv/payments-lab/releases
```

Você deverá observar algo semelhante a:

```text
drwxrws---
```

Observe o:

```text
s
```

na área do grupo.

---

# 35. setgid em diretórios

O número inicial:

```text
2
```

em:

```text
2770
```

ativa o:

```text
setgid
```

no diretório.

Quando utilizado em diretórios compartilhados, novos arquivos tendem a herdar o **grupo proprietário do diretório**.

Isso é muito útil para áreas colaborativas.

---

# 36. Criando um release como Ana

Execute:

```bash
sudo -u ana-lab bash -c \
  'umask 0002; touch /srv/payments-lab/releases/release-v1.txt'
```

Confira:

```bash
ls -l /srv/payments-lab/releases
```

Observe o grupo do arquivo.

Mesmo que Ana possua outro grupo principal, o arquivo deverá pertencer ao grupo:

```text
payments-deploy
```

---

# 37. Bruno pode trabalhar no mesmo arquivo?

Teste:

```bash
sudo -u bruno-lab sh -c \
  'echo "validated by bruno" >> /srv/payments-lab/releases/release-v1.txt'
```

Depois:

```bash
cat /srv/payments-lab/releases/release-v1.txt
```

Como ambos pertencem ao mesmo grupo e o arquivo recebeu permissões adequadas, o trabalho colaborativo é possível.

---

# 38. umask

A `umask` influencia as permissões iniciais concedidas a novos arquivos e diretórios.

Consulte a sua:

```bash
umask
```

Uma saída comum é:

```text
0022
```

Cada processo possui uma umask, normalmente herdada de seu processo pai.

---

# 39. Testando uma umask diferente

Vamos realizar o teste na home de Ana.

Execute:

```bash
sudo -u ana-lab bash -c '
  umask 0027
  touch /home/ana-lab/umask-file.txt
  mkdir /home/ana-lab/umask-dir
'
```

Confira:

```bash
ls -ld /home/ana-lab/umask-file.txt
```

E:

```bash
ls -ld /home/ana-lab/umask-dir
```

Com essa configuração, normalmente teremos:

```text
arquivo     → 640

diretório   → 750
```

A umask funciona como um filtro que remove permissões que não devem ser concedidas por padrão.

---

# 40. Permissão para remover arquivos

Um detalhe importante é que remover um arquivo depende principalmente das permissões do **diretório que contém sua entrada**.

Vamos experimentar.

Crie:

```bash
sudo mkdir /srv/payments-lab/shared-delete
```

Configure:

```bash
sudo chown root:payments-deploy \
  /srv/payments-lab/shared-delete
```

Depois:

```bash
sudo chmod 2770 /srv/payments-lab/shared-delete
```

---

# 41. Criando um arquivo como Ana

Execute:

```bash
sudo -u ana-lab \
  touch /srv/payments-lab/shared-delete/ana.txt
```

Confira:

```bash
ls -l /srv/payments-lab/shared-delete
```

O arquivo pertence a Ana.

Agora execute como Bruno:

```bash
sudo -u bruno-lab \
  rm /srv/payments-lab/shared-delete/ana.txt
```

A remoção deverá funcionar.

Isso pode parecer estranho:

```text
Bruno não era dono do arquivo.
```

Mas ambos possuem escrita no diretório.

---

# 42. Sticky bit

Em alguns diretórios compartilhados, queremos permitir que todos criem arquivos, mas impedir que usuários removam os arquivos uns dos outros.

Crie:

```bash
sudo mkdir /srv/payments-lab/dropbox
```

Configure:

```bash
sudo chown root:payments-deploy \
  /srv/payments-lab/dropbox
```

Depois:

```bash
sudo chmod 1770 /srv/payments-lab/dropbox
```

Observe:

```bash
ls -ld /srv/payments-lab/dropbox
```

O primeiro:

```text
1
```

ativa o **sticky bit**.

---

# 43. Testando o sticky bit

Ana cria um arquivo:

```bash
sudo -u ana-lab \
  touch /srv/payments-lab/dropbox/ana.txt
```

Bruno tenta removê-lo:

```bash
sudo -u bruno-lab \
  rm /srv/payments-lab/dropbox/ana.txt
```

A operação deverá ser negada.

Ana ainda poderá remover seu próprio arquivo:

```bash
sudo -u ana-lab \
  rm /srv/payments-lab/dropbox/ana.txt
```

Esse mecanismo é utilizado, por exemplo, em diretórios compartilhados como:

```text
/tmp
```

Confira:

```bash
ls -ld /tmp
```

---

# 44. root

Consulte:

```bash
id root
```

Você deverá observar:

```text
uid=0(root)
```

O UID:

```text
0
```

representa tradicionalmente o superusuário.

O root possui poderes administrativos amplos sobre o sistema.

Por isso, aplicações e atividades cotidianas não devem utilizar essa identidade sem necessidade.

---

# 45. sudo

O `sudo` permite executar operações privilegiadas de forma controlada.

Por exemplo:

```bash
sudo id
```

A saída deverá indicar:

```text
uid=0(root)
```

Compare com:

```bash
id
```

Sua sessão continua sendo de seu usuário comum.

Apenas aquele comando específico foi executado com privilégio elevado.

---

# 46. Verificando suas permissões sudo

Execute:

```bash
sudo -l
```

O comando mostra quais operações sua conta está autorizada a executar através de `sudo`.

---

# 47. Delegando apenas uma operação

Vamos permitir que Ana execute somente:

```text
/usr/bin/id
```

como root.

Em vez de editar diretamente `/etc/sudoers`, primeiro crie um arquivo temporário no seu diretório pessoal:

```bash
cat > "$HOME/devops-lab-sudoers.tmp" <<'EOF'
ana-lab ALL=(root) NOPASSWD: /usr/bin/id
EOF
```

---

# 48. Validando antes de instalar a regra

Valide a sintaxe **antes** de colocar o arquivo em `/etc/sudoers.d/`:

```bash
sudo visudo -cf "$HOME/devops-lab-sudoers.tmp"
```

Uma configuração válida deverá apresentar algo semelhante a:

```text
parsed OK
```

Somente depois da validação instale a regra com owner e permissões adequados:

```bash
sudo install \
  -o root \
  -g root \
  -m 0440 \
  "$HOME/devops-lab-sudoers.tmp" \
  /etc/sudoers.d/devops-lab
```

Remova o arquivo temporário:

```bash
rm "$HOME/devops-lab-sudoers.tmp"
```

> Validar um arquivo temporário antes de instalá-lo reduz a chance de deixar uma configuração inválida ativa no `sudo`.

---

# 49. Testando o privilégio delegado

Confira exatamente o que Ana pode executar:

```bash
sudo -l -U ana-lab
```

Verifique especificamente se `/usr/bin/id` está autorizado:

```bash
sudo -l -U ana-lab /usr/bin/id
```

Agora execute o comando permitido utilizando a identidade de Ana:

```bash
sudo -u ana-lab sudo /usr/bin/id
```

A saída deverá indicar:

```text
uid=0(root)
```

Para verificar uma operação que **não** foi autorizada, não tente abrir um `sudo` interativo com Ana, porque a conta foi criada sem senha. Consulte a política a partir da conta administrativa:

```bash
sudo -l -U ana-lab /usr/bin/apt
```

Esse comando deve terminar com status diferente de zero e não deve listar `/usr/bin/apt` como autorizado.

Isso confirma a política sem deixar o aluno preso em uma solicitação de senha que a conta `ana-lab` não possui.

---

# 50. O que acabamos de fazer?

Ana recebeu:

```text
permissão administrativa para uma operação
```

e não:

```text
acesso administrativo irrestrito
```

Essa distinção é importante.

Uma política de acesso pode ser pensada como:

```text
identidade
    ↓
recurso/operação
    ↓
permissão necessária
```

em vez de simplesmente:

```text
todo mundo recebe root
```

---

# 51. Gerenciamento de pacotes

Agora vamos observar como o Linux gerencia software instalado.

Em distribuições Debian e Ubuntu, encontramos normalmente duas camadas:

```text
APT
 ↓
dpkg
 ↓
pacotes .deb
```

De forma simplificada:

```text
APT
    localiza pacotes
    consulta repositórios
    resolve dependências
    coordena instalações

dpkg
    trabalha diretamente com pacotes .deb
```

---

# 52. Verificando as ferramentas

Execute:

```bash
apt --version
```

Depois:

```bash
dpkg --version
```

---

# 53. Descobrindo pacotes instalados

Execute:

```bash
dpkg -l | head
```

A lista pode ser bastante grande.

Também podemos consultar um pacote específico:

```bash
dpkg -l bash
```

---

# 54. Descobrindo qual pacote fornece um arquivo

Vamos utilizar o próprio executável do APT, cujo caminho fica em `/usr/bin` nas distribuições alvo desta prática:

```bash
command -v apt
```

Depois:

```bash
dpkg -S "$(command -v apt)"
```

O resultado deverá indicar o pacote responsável por esse arquivo.

> Evitamos usar `command -v bash` neste exemplo porque sistemas em diferentes estágios da migração para `/usr` podem apresentar `/bin/bash` e `/usr/bin/bash` de formas diferentes para o `dpkg`.

---

# 55. Listando arquivos de um pacote

Execute:

```bash
dpkg -L bash | head -n 30
```

O comando mostra arquivos registrados como pertencentes ao pacote.

Isso é útil durante troubleshooting.

Por exemplo:

> Quem instalou este binário?

ou:

> Quais arquivos fazem parte deste pacote?

---

# 56. Informações sobre um pacote

Execute:

```bash
apt show bash
```

Observe informações como:

```text
versão

arquitetura

dependências

descrição
```

---

# 57. Dependências

Consulte:

```bash
apt-cache depends bash
```

Um pacote raramente existe completamente isolado.

Software instalado pode depender de:

```text
bibliotecas

outros pacotes

componentes do sistema
```

Por isso, atualizações precisam considerar a árvore de dependências.

---

# 58. Repositórios

Consulte a política do APT:

```bash
apt-cache policy
```

O APT obtém pacotes através de fontes configuradas no sistema.

Em versões recentes de Ubuntu, essas configurações podem aparecer em:

```text
/etc/apt/sources.list
```

ou:

```text
/etc/apt/sources.list.d/
```

Liste:

```bash
ls /etc/apt/sources.list.d/
```

> Não modifique os repositórios durante esta prática.

---

# 59. Atualizando os metadados dos repositórios

Execute:

```bash
sudo apt update
```

É importante entender o que esse comando faz.

```text
apt update
```

não significa:

```text
atualizar todos os programas
```

Ele atualiza principalmente a informação local sobre os pacotes disponíveis nos repositórios.

### Se `apt update` falhar

Alguns erros dependem do ambiente e não das permissões do laboratório:

- mensagens como `Temporary failure resolving` indicam normalmente problema de rede ou DNS;
- mensagens como `Could not get lock` indicam que outro processo de gerenciamento de pacotes está em execução;
- nesse segundo caso, aguarde o outro processo terminar e tente novamente;
- **não apague manualmente arquivos de lock do APT/DPKG** para tentar forçar a execução.

Se a máquina estiver sem acesso aos repositórios, as partes anteriores do laboratório continuam válidas. Retome a partir da instalação de pacotes quando a conectividade estiver disponível.

---

# 60. Pacotes que possuem atualização disponível

Execute:

```bash
apt list --upgradable
```

Dependendo de quando sua máquina foi atualizada, a lista pode estar vazia ou conter vários pacotes.

---

# 61. Simulando uma atualização

Antes de modificar o sistema, podemos pedir ao APT para simular uma operação.

Execute:

```bash
sudo apt -s upgrade
```

Observe atentamente informações como:

```text
pacotes que seriam atualizados

novos pacotes necessários como dependência

pacotes que permaneceriam na versão atual
```

Nenhuma alteração real deverá ser feita.

> `apt upgrade` não remove pacotes já instalados para concluir uma atualização. Quando uma mudança exigiria remoção de pacotes, o comportamento de `full-upgrade` é diferente. Nesta prática utilizaremos apenas a simulação.

Esse tipo de inspeção é útil antes de mudanças importantes.

---

# 62. Instalando um pacote

Vamos instalar as ferramentas de ACL que serão utilizadas na próxima parte da prática.

Execute:

```bash
sudo apt install acl
```

Se o pacote já estiver instalado, o APT apenas informará que a versão adequada já está disponível.

Teste:

```bash
setfacl --version
getfacl --version
```

---

# 63. Descobrindo qual pacote fornece um executável

Descubra:

```bash
command -v setfacl
```

Depois:

```bash
dpkg -S "$(command -v setfacl)"
```

Você deverá encontrar o pacote:

```text
acl
```

---

# 64. Informações sobre o pacote instalado

Execute:

```bash
dpkg -l acl
```

Depois:

```bash
apt show acl
```

Também:

```bash
dpkg -L acl
```

Temos agora a relação:

```text
repositório
     ↓
APT
     ↓
pacote
     ↓
arquivos instalados
```

---

# 65. Simulando a remoção

Antes de remover qualquer pacote, podemos visualizar o que aconteceria.

Execute:

```bash
sudo apt -s remove acl
```

Observe as ações propostas.

Não remova o pacote agora, pois ele será utilizado na próxima seção.

---

# 66. ACLs — um exemplo de controle mais específico

O modelo tradicional trabalha principalmente com:

```text
owner

group

others
```

Em alguns casos queremos conceder uma permissão específica sem reorganizar toda a estrutura de grupos.

Para isso existem ACLs.

Confirme que as ferramentas instaladas na etapa anterior estão disponíveis:

```bash
command -v setfacl
command -v getfacl
```

> Em uma VM Ubuntu/Debian comum, o filesystem utilizado normalmente suporta POSIX ACLs. Se `setfacl` retornar uma mensagem indicando que a operação não é suportada, utilize uma VM/filesystem Linux convencional para esta parte da prática.

---

# 67. Preparando um arquivo para ACL

Crie:

```bash
sudo tee /srv/payments-lab/config/report.txt > /dev/null <<'EOF'
internal report
EOF
```

Configure:

```bash
sudo chown payments-svc:payments-svc \
  /srv/payments-lab/config/report.txt
```

Depois:

```bash
sudo chmod 640 /srv/payments-lab/config/report.txt
```

---

# 68. Bruno consegue ler?

Execute:

```bash
sudo -u bruno-lab \
  cat /srv/payments-lab/config/report.txt
```

A leitura deverá ser negada.

---

# 69. Concedendo acesso específico com ACL

Execute:

```bash
sudo setfacl \
  -m u:bruno-lab:r \
  /srv/payments-lab/config/report.txt
```

Confira:

```bash
getfacl -p /srv/payments-lab/config/report.txt
```

Agora teste:

```bash
sudo -u bruno-lab \
  cat /srv/payments-lab/config/report.txt
```

Bruno deverá conseguir ler o arquivo.

---

# 70. Observando a indicação de ACL

Execute:

```bash
ls -l /srv/payments-lab/config/report.txt
```

Você poderá observar:

```text
+
```

ao lado das permissões.

Isso indica a existência de informações adicionais de ACL.

ACLs são úteis, mas também aumentam a complexidade das regras de acesso.

---

# 71. Laboratório — Corrigindo permissões de uma aplicação

Considere o seguinte cenário:

> Uma aplicação chamada `payments-api` está sendo preparada para produção. O serviço deve executar como `payments-svc`. Ana e Bruno fazem parte da equipe responsável por deployments.

As seguintes regras precisam ser implementadas:

1. a aplicação não deve executar como root;
2. Ana e Bruno devem conseguir publicar arquivos no diretório de releases;
3. novos arquivos de release devem permanecer associados ao grupo `payments-deploy`;
4. membros da equipe devem conseguir ler `app.env`;
5. membros da equipe não devem conseguir alterar `app.env`;
6. somente `payments-svc` deve conseguir ler `secret.env`;
7. a aplicação deve possuir seu diretório de logs;
8. Ana deve possuir somente o privilégio sudo explicitamente configurado anteriormente;
9. ninguém deve simplesmente utilizar permissões abertas como `777` para solucionar o problema.

---

## Etapa 1 — Identidade da aplicação

Descubra:

```text
UID

GID

shell
```

de:

```text
payments-svc
```

Utilize:

```text
id

getent
```

---

## Etapa 2 — Grupo de deployment

Verifique se:

```text
ana-lab

bruno-lab
```

participam de:

```text
payments-deploy
```

---

## Etapa 3 — Diretório de releases

Configure:

```text
owner → payments-svc

group → payments-deploy
```

Membros do grupo devem possuir:

```text
read
write
execute
```

Novos arquivos também devem herdar:

```text
payments-deploy
```

---

## Etapa 4 — Configuração da aplicação

Configure:

```text
/srv/payments-lab/config/app.env
```

para:

```text
owner → leitura e escrita

group → leitura

others → nenhum acesso
```

---

## Etapa 5 — Segredo

Configure:

```text
/srv/payments-lab/config/secret.env
```

para que somente:

```text
payments-svc
```

possa acessá-lo.

---

## Etapa 6 — Logs

Garanta que:

```text
payments-svc
```

seja proprietário de:

```text
/srv/payments-lab/logs
```

---

## Etapa 7 — Validação

Teste explicitamente:

```text
Ana consegue ler app.env?

Ana consegue alterar app.env?

Ana consegue ler secret.env?

payments-svc consegue ler secret.env?

Ana consegue criar releases?

Bruno consegue modificar um release compartilhado?
```

Não considere a configuração concluída sem realizar os testes.

---

# 72. Solução do laboratório

<details>

<summary>Mostrar solução</summary>

### Identidade da aplicação

```bash
id payments-svc
```

```bash
getent passwd payments-svc
```

---

### Verificando usuários do deployment

```bash
id ana-lab
```

```bash
id bruno-lab
```

```bash
getent group payments-deploy
```

---

### Diretório de releases

```bash
sudo chown payments-svc:payments-deploy \
  /srv/payments-lab/releases
```

```bash
sudo chmod 2770 \
  /srv/payments-lab/releases
```

---

### Configuração da aplicação

```bash
sudo chown payments-svc:payments-deploy \
  /srv/payments-lab/config/app.env
```

```bash
sudo chmod 640 \
  /srv/payments-lab/config/app.env
```

---

### Segredo

```bash
sudo chown payments-svc:payments-svc \
  /srv/payments-lab/config/secret.env
```

```bash
sudo chmod 600 \
  /srv/payments-lab/config/secret.env
```

---

### Logs

```bash
sudo chown -R payments-svc:payments-svc \
  /srv/payments-lab/logs
```

---

### Ana lê a configuração

```bash
sudo -u ana-lab \
  cat /srv/payments-lab/config/app.env
```

Deve funcionar.

---

### Ana tenta modificar a configuração

```bash
sudo -u ana-lab sh -c \
  'echo "TEST=true" >> /srv/payments-lab/config/app.env'
```

Deve falhar.

---

### Ana tenta ler o segredo

```bash
sudo -u ana-lab \
  cat /srv/payments-lab/config/secret.env
```

Deve falhar.

---

### Aplicação lê o segredo

```bash
sudo -u payments-svc \
  cat /srv/payments-lab/config/secret.env
```

Deve funcionar.

---

### Ana cria um release

```bash
sudo -u ana-lab bash -c \
  'umask 0002; touch /srv/payments-lab/releases/release-v2.txt'
```

---

### Conferindo grupo do release

```bash
ls -l /srv/payments-lab/releases
```

O arquivo deverá utilizar:

```text
payments-deploy
```

como grupo.

---

### Bruno modifica o release

```bash
sudo -u bruno-lab sh -c \
  'echo "reviewed" >> /srv/payments-lab/releases/release-v2.txt'
```

</details>

---

# 73. Desafio

Sem consultar imediatamente a solução, realize as seguintes tarefas.

1. descubra o UID de `ana-lab`;
2. descubra o GID principal de `bruno-lab`;
3. liste todos os grupos de `ana-lab`;
4. descubra qual shell está configurado para `payments-svc`;
5. descubra quem é owner e group de `app.env`;
6. escreva as permissões `640` utilizando `r`, `w` e `x`;
7. escreva as permissões `750` utilizando `r`, `w` e `x`;
8. descubra se `ana-lab` consegue ler `app.env`;
9. descubra se `ana-lab` consegue escrever em `app.env`;
10. descubra se `payments-svc` consegue ler `secret.env`;
11. descubra qual pacote fornece o executável `apt`;
12. liste os primeiros 20 arquivos pertencentes ao pacote `bash`;
13. descubra quais pacotes possuem atualização disponível;
14. simule uma atualização dos pacotes sem alterar o sistema;
15. descubra qual pacote fornece o executável `setfacl`.

---

# 74. Solução do desafio

<details>

<summary>Mostrar solução</summary>

### 1. UID de Ana

```bash
id -u ana-lab
```

---

### 2. GID principal de Bruno

```bash
id -g bruno-lab
```

---

### 3. Grupos de Ana

```bash
id -Gn ana-lab
```

---

### 4. Shell de payments-svc

```bash
getent passwd payments-svc
```

---

### 5. Owner e group de app.env

```bash
ls -l /srv/payments-lab/config/app.env
```

ou:

```bash
stat /srv/payments-lab/config/app.env
```

---

### 6. Permissão 640

```text
rw-r-----
```

---

### 7. Permissão 750

```text
rwxr-x---
```

---

### 8. Testar leitura como Ana

```bash
sudo -u ana-lab \
  cat /srv/payments-lab/config/app.env
```

---

### 9. Testar escrita como Ana

```bash
sudo -u ana-lab sh -c \
  'echo "TEST=true" >> /srv/payments-lab/config/app.env'
```

---

### 10. Testar segredo como serviço

```bash
sudo -u payments-svc \
  cat /srv/payments-lab/config/secret.env
```

---

### 11. Pacote responsável pelo APT

```bash
dpkg -S "$(command -v apt)"
```

---

### 12. Arquivos do pacote Bash

```bash
dpkg -L bash | head -n 20
```

---

### 13. Atualizações disponíveis

```bash
apt list --upgradable
```

---

### 14. Simular atualização

```bash
sudo apt -s upgrade
```

---

### 15. Pacote responsável pelo setfacl

```bash
dpkg -S "$(command -v setfacl)"
```

</details>

---

# 75. Um exemplo próximo da rotina DevOps

Imagine que uma aplicação apresente:

```text
Permission denied
```

durante um deployment.

Uma reação ruim seria:

```text
dar permissão total para todo mundo
```

Em vez disso, investigue progressivamente.

Primeiro descubra a identidade:

```bash
id
```

ou, para um serviço:

```bash
id payments-svc
```

Depois examine o caminho:

```bash
ls -ld /srv/payments-lab
```

```bash
ls -ld /srv/payments-lab/releases
```

Depois examine o arquivo:

```bash
ls -l /srv/payments-lab/config/app.env
```

Se necessário:

```bash
stat /srv/payments-lab/config/app.env
```

E, caso existam ACLs:

```bash
getfacl -p /srv/payments-lab/config/app.env
```

O raciocínio deve ser:

```text
Qual identidade está executando?
            ↓
Qual recurso está sendo acessado?
            ↓
Quem é owner?
            ↓
Qual é o group?
            ↓
Quais permissões se aplicam?
            ↓
Os diretórios do caminho podem ser atravessados?
            ↓
Existe alguma ACL adicional?
```

Esse processo é muito mais seguro do que simplesmente aumentar permissões até o erro desaparecer.

---

# 76. Comandos utilizados nesta prática

| Comando | Finalidade |
| --- | --- |
| `id` | Mostrar UID, GID e grupos |
| `groups` | Mostrar grupos do usuário |
| `getent` | Consultar bases de usuários e grupos |
| `adduser` | Criar usuário |
| `useradd` | Criar usuário ou conta de serviço |
| `usermod` | Modificar usuário |
| `groupadd` | Criar grupo |
| `chown` | Alterar usuário e grupo proprietário |
| `chmod` | Alterar permissões |
| `umask` | Controlar permissões iniciais |
| `stat` | Exibir metadados |
| `sudo` | Executar operação com outra identidade |
| `visudo` | Editar ou validar configuração do sudo |
| `apt` | Gerenciamento de pacotes em alto nível |
| `apt-cache` | Consultar metadados do APT |
| `dpkg` | Gerenciar e consultar pacotes `.deb` |
| `setfacl` | Alterar ACL |
| `getfacl` | Visualizar ACL |

---

# 77. Permissões importantes

| Permissão | Arquivo | Diretório |
| --- | --- | --- |
| `r` | Ler conteúdo | Listar entradas |
| `w` | Modificar conteúdo | Criar, remover e renomear entradas, normalmente em conjunto com `x` |
| `x` | Executar | Atravessar/acessar caminhos internos |

---

# 78. Valores numéricos

| Valor | Permissão |
| --- | --- |
| `0` | `---` |
| `1` | `--x` |
| `2` | `-w-` |
| `3` | `-wx` |
| `4` | `r--` |
| `5` | `r-x` |
| `6` | `rw-` |
| `7` | `rwx` |

Exemplos:

```text
600 → rw-------

640 → rw-r-----

644 → rw-r--r--

750 → rwxr-x---

755 → rwxr-xr-x
```

---

# 79. Bits especiais apresentados

| Valor inicial | Nome | Uso comum |
| --- | --- | --- |
| `2` | setgid | Herdar grupo em diretórios compartilhados |
| `1` | sticky bit | Impedir remoção de arquivos de outros usuários |

Exemplos:

```text
2770
```

Diretório colaborativo com `setgid`.

```text
1777
```

Diretório compartilhado com sticky bit, como normalmente ocorre em `/tmp`.

---

# 80. Boas práticas

Evite utilizar permissões excessivas apenas para remover rapidamente um erro de acesso.

Em especial, não transforme:

```text
chmod 777
```

em uma solução genérica.

Antes de alterar permissões, descubra:

```text
qual processo está falhando

↓

com qual usuário ele executa

↓

qual arquivo ou diretório está sendo acessado

↓

quem é owner

↓

qual é o group

↓

quais permissões realmente são necessárias
```

Também evite:

```text
executar aplicações como root sem necessidade

compartilhar contas administrativas

dar sudo irrestrito para automações

modificar /etc/sudoers sem validação

instalar pacotes de fontes desconhecidas

executar atualizações de produção sem avaliar as mudanças
```

---

# 81. Limpando o laboratório

Antes de remover qualquer coisa, confira o caminho:

```bash
ls -ld /srv/payments-lab
```

Verifique a estrutura utilizando uma ferramenta que já faz parte do laboratório anterior:

```bash
find /srv/payments-lab -maxdepth 3 -print
```

Remova a configuração criada para `sudo`, caso exista:

```bash
sudo rm -f /etc/sudoers.d/devops-lab
```

Remova a aplicação somente depois de confirmar o caminho:

```bash
sudo rm -rf -- /srv/payments-lab
```

> O caminho acima é específico deste laboratório. Não substitua por um caminho genérico e sempre confira antes de executar `rm -rf`.

Remova os usuários humanos somente se eles existirem:

```bash
getent passwd ana-lab >/dev/null && sudo userdel -r ana-lab
getent passwd bruno-lab >/dev/null && sudo userdel -r bruno-lab
```

Remova a conta de serviço somente se ela existir:

```bash
getent passwd payments-svc >/dev/null && sudo userdel payments-svc
```

Remova o grupo de deployment somente se ele existir:

```bash
getent group payments-deploy >/dev/null && sudo groupdel payments-deploy
```

Caso o grupo privado de `payments-svc` ainda exista:

```bash
getent group payments-svc >/dev/null && sudo groupdel payments-svc
```

---

## Sobre o pacote acl

Se `acl` já estava instalado antes do laboratório, mantenha-o.

Se você confirmou que ele foi instalado exclusivamente durante esta prática e deseja removê-lo **depois de concluir a seção de ACLs**:

```bash
sudo apt remove acl
```

Não remova pacotes do sistema apenas para "limpar" o laboratório sem antes verificar se eles já eram utilizados.

---

# Checklist

Ao concluir a prática, verifique se você consegue realizar as seguintes tarefas sem consultar diretamente a solução:

- diferenciar UID de username;
- diferenciar GID de group name;
- consultar informações com `id`;
- consultar usuários e grupos com `getent`;
- explicar o papel de `/etc/passwd`;
- explicar por que `/etc/shadow` possui acesso restrito;
- identificar o shell associado a uma conta;
- diferenciar conta humana de conta de serviço;
- adicionar usuários a grupos;
- identificar owner e group de um arquivo;
- alterar ownership com `chown`;
- interpretar `rwx`;
- interpretar permissões numéricas;
- utilizar `chmod`;
- explicar a diferença entre `rwx` em arquivos e diretórios;
- utilizar `umask`;
- explicar o uso de setgid em diretórios compartilhados;
- explicar o sticky bit;
- testar permissões utilizando outra identidade;
- compreender por que `sudo` é preferível a trabalhar permanentemente como root;
- consultar permissões sudo;
- utilizar `visudo`;
- diferenciar `apt` de `dpkg`;
- descobrir qual pacote instalou determinado arquivo;
- listar arquivos pertencentes a um pacote;
- consultar dependências;
- executar `apt update`;
- listar atualizações disponíveis;
- simular uma atualização antes de aplicá-la;
- instalar um pacote;
- identificar um problema de permissão sem recorrer imediatamente a permissões excessivas.

---

# Próxima prática

Na próxima aula serão explorados:

**Processos**

**PID e PPID**

**Estados de processos**

**Sinais**

**ps**

**top**

**systemd**

**systemctl**

**journalctl**

Esses conceitos serão utilizados para compreender como aplicações e serviços são executados, supervisionados e diagnosticados em sistemas Linux.

---

# Referências

**NEMETH, E.; SNYDER, G.; HEIN, T. R.; WHALEY, B.; MACKIN, D.**

*UNIX and Linux System Administration Handbook*. 5. ed. Pearson, 2018.

Capítulos utilizados:

- **Cap. 3 — Access Control and Rootly Powers**
- **Cap. 5 — The Filesystem**
- **Cap. 6 — Software Installation and Management**
- **Cap. 8 — User Management**

**KIM, G.; HUMBLE, J.; DEBOIS, P.; WILLIS, J.; FORSGREN, N.**

*The DevOps Handbook: How to Create World-Class Agility, Reliability, & Security in Technology Organizations*. 2. ed. IT Revolution Press, 2021.

Conteúdo complementar:

- **Parte VI, Cap. 22 — Information Security Is Everyone’s Job Every Day**
- **Parte VI, Cap. 23 — Protecting the Deployment Pipeline**