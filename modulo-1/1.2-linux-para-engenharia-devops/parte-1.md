# 1.2 — Linux para Engenharia DevOps

## Prática 01 — Terminal, Filesystem, Pipes e Redirecionamento

Este material complementa a aula teórica **Linux para Engenharia DevOps**.

O objetivo não é memorizar comandos, mas desenvolver familiaridade com o ambiente Linux e compreender como pequenas ferramentas podem ser combinadas para investigar e manipular informações.

> Os exemplos foram pensados para distribuições Linux como Ubuntu e Debian. A maior parte dos comandos também está disponível em outras distribuições.

---

## Objetivos

Ao final desta prática, você deverá ser capaz de:

* identificar informações básicas sobre o sistema Linux;
* navegar pelo filesystem;
* trabalhar com caminhos absolutos e relativos;
* criar, copiar, mover e remover arquivos e diretórios;
* visualizar e inspecionar arquivos;
* localizar arquivos e pesquisar conteúdo;
* compreender `stdin`, `stdout` e `stderr`;
* redirecionar entrada e saída;
* combinar programas utilizando pipes;
* utilizar filtros comuns do ambiente Unix/Linux;
* aplicar esses recursos em uma pequena investigação de logs.

---

# 1. Conhecendo o ambiente

Antes de começar a administrar um sistema Linux, é importante entender **em qual ambiente estamos trabalhando**.

## Usuário atual

```bash
whoami
```

Exibe o usuário que está executando os comandos.

---

## Nome da máquina

```bash
hostname
```

Em ambientes reais, isso ajuda a identificar **em qual servidor estamos conectados**.

Isso é especialmente importante quando existem várias máquinas:

```text
api-01
api-02
database-01
worker-01
```

Executar um comando no servidor errado pode causar problemas graves.

---

## Kernel

```bash
uname -r
```

Exibe a versão do kernel Linux em execução.

Para visualizar mais informações:

```bash
uname -a
```

---

## Arquitetura

```bash
uname -m
```

Exemplos comuns:

```text
x86_64
aarch64
```

`x86_64` normalmente representa máquinas Intel/AMD de 64 bits.

`aarch64` representa a arquitetura ARM de 64 bits.

Essa diferença aparece, por exemplo, ao selecionar imagens Docker ou binários de ferramentas.

---

## Distribuição Linux

```bash
cat /etc/os-release
```

Exemplo:

```text
NAME="Ubuntu"
VERSION="24.04 LTS"
ID=ubuntu
```

Lembre-se:

```text
Linux        → kernel

Ubuntu       → distribuição

Debian       → distribuição

Fedora       → distribuição
```

---

# 2. Descobrindo qual shell estamos utilizando

O shell interpreta os comandos digitados no terminal.

A variável `SHELL` normalmente informa o **shell de login configurado para o usuário**:

```bash
echo "$SHELL"
```

Uma saída comum é:

```text
/bin/bash
```

Isso não garante, porém, que esse seja o shell que está executando a sessão atual. Para observar o processo de shell associado à sessão atual, execute:

```bash
ps -p $$ -o comm=
```

Em uma sessão Bash, uma saída comum é:

```text
bash
```

Verifique a versão do Bash instalada:

```bash
bash --version
```

Neste curso utilizaremos principalmente **Bash**.

---

# 3. Obtendo ajuda

Não é necessário memorizar todas as opções de todos os comandos.

Saber consultar documentação é uma habilidade muito mais importante.

## `--help`

Muitos programas possuem ajuda integrada:

```bash
ls --help
```

```bash
grep --help
```

```bash
cp --help
```

---

## Manual do sistema

Linux possui páginas de manual acessíveis através do comando `man`.

Exemplo:

```bash
man ls
```

Navegação básica dentro do manual:

```text
↑ ↓        navegar

SPACE      próxima página

/termo     pesquisar

q          sair
```

Experimente:

```bash
man grep
```

Depois pesquise:

```text
ignore case
```

---

# 4. Conhecendo o filesystem

No Linux, o filesystem é organizado como uma única árvore.

O ponto inicial dessa árvore é:

```text
/
```

Liste seu conteúdo:

```bash
ls /
```

Você deverá encontrar diretórios semelhantes a:

```text
bin
boot
dev
etc
home
proc
sys
tmp
usr
var
```

---

## Explorando alguns diretórios

### Configurações

```bash
ls /etc
```

Muitas configurações do sistema e de aplicações ficam em:

```text
/etc
```

---

### Dados variáveis

```bash
ls /var
```

É comum encontrar:

```text
cache
lib
log
tmp
```

Logs tradicionalmente ficam em:

```text
/var/log
```

Experimente:

```bash
ls /var/log
```

---

### Usuários

```bash
ls /home
```

Os diretórios pessoais dos usuários normalmente ficam dentro de `/home`.

---

## Informações expostas pelo kernel

Alguns diretórios representam informações fornecidas dinamicamente pelo próprio sistema.

Experimente:

```bash
head /proc/meminfo
```

Também:

```bash
cat /proc/uptime
```

E:

```bash
ls /sys/class/net
```

A última instrução normalmente mostra interfaces de rede, como:

```text
eth0
lo
```

Isso demonstra uma característica importante dos sistemas Unix/Linux:

> Muitos recursos do sistema são apresentados através da interface do filesystem.

---

# 5. Diretório atual

Todo shell possui um **diretório de trabalho atual**.

Para descobrir onde estamos:

```bash
pwd
```

`pwd` significa:

```text
Print Working Directory
```

Exemplo:

```text
/home/aluno
```

---

# 6. Listando arquivos e diretórios

Utilize:

```bash
ls
```

Para obter mais detalhes:

```bash
ls -l
```

Para incluir arquivos ocultos:

```bash
ls -a
```

Uma combinação muito comum é:

```bash
ls -lah
```

Onde:

```text
-l   exibição detalhada

-a   inclui arquivos ocultos

-h   tamanhos legíveis por humanos
```

Por enquanto, não é necessário compreender completamente as informações de permissões exibidas por `ls -l`.

Esse assunto será tratado na próxima aula.

---

# 7. Arquivos ocultos

No Linux, arquivos cujo nome começa com `.` são considerados ocultos.

Exemplo:

```text
.bashrc
.profile
.gitconfig
```

Compare:

```bash
ls
```

com:

```bash
ls -a
```

Arquivos ocultos são muito utilizados para armazenar configurações.

---

# 8. Navegação

Utilizamos `cd` para mudar de diretório.

```bash
cd /var
```

Verifique:

```bash
pwd
```

Resultado:

```text
/var
```

Entre em:

```bash
cd /var/log
```

E novamente:

```bash
pwd
```

---

# 9. Caminho absoluto

Um caminho absoluto começa na raiz `/`.

Exemplo:

```text
/var/log
```

```text
/etc
```

```text
/home/aluno
```

Podemos navegar diretamente utilizando:

```bash
cd /var/log
```

Independentemente de onde estivermos.

---

# 10. Caminho relativo

Um caminho relativo é interpretado a partir do diretório atual.

Suponha que você esteja em:

```text
/var
```

Você pode executar:

```bash
cd log
```

Em vez de:

```bash
cd /var/log
```

---

# 11. Diretório atual e diretório pai

O símbolo:

```text
.
```

representa o diretório atual.

O símbolo:

```text
..
```

representa o diretório pai.

Exemplo:

```bash
cd /var/log
```

Depois:

```bash
cd ..
```

Agora:

```bash
pwd
```

Resultado:

```text
/var
```

---

## Voltando para o diretório home

Você pode utilizar:

```bash
cd ~
```

ou simplesmente:

```bash
cd
```

Confira:

```bash
pwd
```

---

# 12. Criando nosso ambiente de laboratório

Para evitar modificar arquivos importantes do sistema, toda a prática será feita dentro do diretório pessoal do usuário.

Crie:

```bash
mkdir ~/devops-linux-lab
```

> Se esse diretório já existir por causa de uma execução anterior da prática, utilize um diretório novo ou limpe conscientemente o laboratório anterior antes de continuar. Isso evita que arquivos antigos alterem os resultados esperados.

Entre nele:

```bash
cd ~/devops-linux-lab
```

Confira:

```bash
pwd
```

---

# 13. Criando diretórios

Crie:

```bash
mkdir config
mkdir logs
mkdir data
mkdir backup
```

Verifique:

```bash
ls
```

Resultado esperado:

```text
backup
config
data
logs
```

---

## Criando estruturas completas

A opção `-p` permite criar diretórios intermediários.

```bash
mkdir -p apps/api/config
```

Verifique:

```bash
find apps
```

Resultado semelhante a:

```text
apps
apps/api
apps/api/config
```

---

# 14. Criando arquivos

Utilize:

```bash
touch config/app.conf
```

Verifique:

```bash
ls config
```

Podemos criar vários arquivos:

```bash
touch logs/application.log
touch data/users.csv
```

---

# 15. Escrevendo conteúdo em arquivos

Uma forma simples de produzir texto no terminal é:

```bash
echo "Hello Linux"
```

A saída será enviada para o terminal:

```text
Hello Linux
```

Mais adiante veremos como enviar essa saída para arquivos.

---

# 16. Preparando os arquivos do laboratório

Vamos adicionar alguns dados para utilizar durante os exercícios.

> Os comandos desta seção servem apenas para preparar o laboratório. Alguns recursos de Bash utilizados aqui serão explicados em uma aula posterior.

## Configuração da aplicação

Execute:

```bash
cat > config/app.conf <<'EOF'
APP_NAME=payments-api
APP_ENV=development
PORT=8080
LOG_LEVEL=INFO
EOF
```

---

## Usuários

```bash
cat > data/users.csv <<'EOF'
id,name,team
1,Ana,backend
2,Carlos,frontend
3,Marina,backend
4,Pedro,devops
5,Julia,devops
EOF
```

---

## Logs

```bash
cat > logs/application.log <<'EOF'
2026-09-10T10:00:01Z INFO api /health 200 12ms
2026-09-10T10:00:05Z INFO api /users 200 35ms
2026-09-10T10:00:08Z WARN api /users 429 10ms
2026-09-10T10:00:11Z INFO worker email-job 200 82ms
2026-09-10T10:00:15Z ERROR api /orders 500 125ms
2026-09-10T10:00:18Z INFO api /products 200 24ms
2026-09-10T10:00:20Z ERROR worker payment-job 500 340ms
2026-09-10T10:00:22Z INFO api /health 200 8ms
2026-09-10T10:00:26Z WARN api /login 401 18ms
2026-09-10T10:00:30Z ERROR api /checkout 500 210ms
EOF
```

Confira a estrutura:

```bash
find .
```

---

# 17. Visualizando arquivos

## `cat`

Para visualizar todo o conteúdo:

```bash
cat config/app.conf
```

Também:

```bash
cat data/users.csv
```

`cat` funciona bem para arquivos pequenos.

Evite utilizá-lo para imprimir arquivos gigantes no terminal.

---

# 18. Visualizando arquivos maiores

Para arquivos maiores, podemos utilizar:

```bash
less logs/application.log
```

Dentro do `less`:

```text
↑ ↓        navegar

SPACE      avançar

/ERROR     pesquisar ERROR

q          sair
```

---

# 19. Início de um arquivo

Utilize:

```bash
head logs/application.log
```

Podemos escolher quantas linhas queremos:

```bash
head -n 3 logs/application.log
```

Resultado:

```text
2026-09-10T10:00:01Z INFO api /health 200 12ms
2026-09-10T10:00:05Z INFO api /users 200 35ms
2026-09-10T10:00:08Z WARN api /users 429 10ms
```

---

# 20. Final de um arquivo

Utilize:

```bash
tail logs/application.log
```

Ou:

```bash
tail -n 3 logs/application.log
```

Isso é especialmente útil para logs, porque normalmente estamos interessados nos eventos mais recentes.

---

# 21. Acompanhando um arquivo em tempo real

Uma operação extremamente comum durante troubleshooting é:

```bash
tail -f logs/application.log
```

O terminal permanecerá acompanhando novas linhas adicionadas ao arquivo.

Abra outro terminal e execute:

```bash
cd ~/devops-linux-lab
```

Depois:

```bash
echo "2026-09-10T10:01:00Z ERROR api /payment 500 500ms" >> logs/application.log
```

O primeiro terminal deverá mostrar a nova entrada imediatamente.

Para interromper:

```text
Ctrl + C
```

---

# 22. Copiando arquivos

Copie:

```bash
cp config/app.conf backup/app.conf
```

Verifique:

```bash
ls backup
```

---

## Copiando diretórios

Para copiar um diretório completo:

```bash
cp -r config backup/config
```

Confira:

```bash
find backup
```

---

# 23. Movendo arquivos

Crie:

```bash
touch config/database.conf
```

Depois mova:

```bash
mv config/database.conf backup/database.conf
```

Confira:

```bash
ls config
```

```bash
ls backup
```

---

# 24. Renomeando arquivos

No Linux, renomear e mover são operações realizadas pelo mesmo comando.

```bash
mv backup/database.conf backup/database.conf.old
```

Confira:

```bash
ls backup
```

---

# 25. Removendo arquivos

Crie um arquivo temporário:

```bash
touch arquivo-temporario.txt
```

Verifique:

```bash
ls
```

Agora remova:

```bash
rm arquivo-temporario.txt
```

Confira novamente:

```bash
ls
```

> `rm` normalmente não envia arquivos para uma lixeira. Em servidores, uma remoção incorreta pode ser definitiva.

---

# 26. Removendo diretórios

Crie:

```bash
mkdir teste
```

Para remover um diretório vazio:

```bash
rmdir teste
```

Se o diretório possuir arquivos, normalmente será necessário utilizar uma operação recursiva. Crie novamente um diretório de teste com um arquivo dentro:

```bash
mkdir teste
touch teste/exemplo.txt
```

Agora remova o diretório e seu conteúdo:

```bash
rm -r teste
```

Tenha cuidado com `rm -r`, especialmente quando executado com privilégios administrativos.

---

# 27. Descobrindo o tipo de um arquivo

Utilize:

```bash
file config/app.conf
```

Também:

```bash
file /bin/bash
```

E:

```bash
file /dev/null
```

Observe que diferentes recursos podem aparecer através do filesystem.

---

# 28. Inspecionando metadados

Utilize:

```bash
stat config/app.conf
```

O comando mostra informações como:

```text
tamanho
datas
identificador
proprietário
permissões
```

Não vamos aprofundar permissões nesta prática.

---

# 29. Links simbólicos

Crie:

```bash
ln -s config/app.conf app.conf
```

Agora:

```bash
ls -l
```

Você deverá observar algo semelhante a:

```text
app.conf -> config/app.conf
```

Visualize:

```bash
cat app.conf
```

O conteúdo exibido vem de:

```text
config/app.conf
```

---

## Testando o comportamento do link

Renomeie o arquivo original:

```bash
mv config/app.conf config/app.conf.old
```

Agora tente:

```bash
cat app.conf
```

O link não consegue mais encontrar seu destino.

Restaure:

```bash
mv config/app.conf.old config/app.conf
```

Teste novamente:

```bash
cat app.conf
```

---

# 30. Localizando arquivos

Uma ferramenta importante para localizar recursos no filesystem é:

```bash
find
```

Liste tudo dentro do laboratório:

```bash
find .
```

---

## Procurando arquivos por nome

```bash
find . -name "app.conf"
```

---

## Procurando arquivos `.log`

```bash
find . -name "*.log"
```

---

## Procurando apenas arquivos

```bash
find . -type f
```

---

## Procurando apenas diretórios

```bash
find . -type d
```

---

# 31. Pesquisando conteúdo com `grep`

`grep` procura padrões dentro de texto.

Procure:

```bash
grep "ERROR" logs/application.log
```

Resultado:

```text
2026-09-10T10:00:15Z ERROR api /orders 500 125ms
2026-09-10T10:00:20Z ERROR worker payment-job 500 340ms
2026-09-10T10:00:30Z ERROR api /checkout 500 210ms
2026-09-10T10:01:00Z ERROR api /payment 500 500ms
```

---

## Pesquisa sem diferenciar maiúsculas e minúsculas

```bash
grep -i "error" logs/application.log
```

---

## Exibindo o número da linha

```bash
grep -n "ERROR" logs/application.log
```

---

## Excluindo um padrão

Podemos mostrar todas as linhas que **não** contêm determinado valor:

```bash
grep -v "INFO" logs/application.log
```

Isso exibirá principalmente eventos `WARN` e `ERROR`.

---

# 32. Contando linhas

Utilize:

```bash
wc -l logs/application.log
```

`wc` pode contar diferentes unidades. Algumas opções importantes são:

```text
-l   linhas
-w   palavras
-c   bytes
-m   caracteres
```

Sem opções, `wc` normalmente apresenta **linhas, palavras e bytes**.

Exemplo:

```bash
wc logs/application.log
```

---

# 33. Entrada, saída e erro

Quando executamos um programa, ele normalmente possui três fluxos padrão:

```text
stdin   → 0
stdout  → 1
stderr  → 2
```

De forma simplificada:

```text
             ┌──────────────┐
stdin ──────▶│   Programa   │──────▶ stdout
             │              │
             └──────────────┘──────▶ stderr
```

Por padrão, esses fluxos estão normalmente ligados ao terminal.

---

# 34. stdout

Execute:

```bash
echo "Hello DevOps"
```

O texto:

```text
Hello DevOps
```

é enviado para a saída padrão:

```text
stdout
```

---

# 35. stderr

Execute:

```bash
ls arquivo-que-nao-existe
```

O sistema apresentará um erro semelhante a:

```text
ls: cannot access 'arquivo-que-nao-existe': No such file or directory
```

Essa mensagem é enviada para:

```text
stderr
```

e não para `stdout`.

Essa distinção será importante para automações.

---

# 36. Redirecionando stdout

Utilizamos:

```text
>
```

para enviar a saída para um arquivo.

Execute:

```bash
echo "Minha primeira saída" > output.txt
```

Confira:

```bash
cat output.txt
```

---

# 37. Atenção ao `>`

Execute novamente:

```bash
echo "Nova saída" > output.txt
```

Depois:

```bash
cat output.txt
```

Resultado:

```text
Nova saída
```

O conteúdo anterior foi substituído.

Portanto:

```text
>
```

**sobrescreve o arquivo**.

---

# 38. Acrescentando conteúdo

Para acrescentar dados sem apagar o conteúdo existente:

```text
>>
```

Execute:

```bash
echo "Linha 1" > output.txt
```

Depois:

```bash
echo "Linha 2" >> output.txt
```

E:

```bash
echo "Linha 3" >> output.txt
```

Confira:

```bash
cat output.txt
```

Resultado:

```text
Linha 1
Linha 2
Linha 3
```

---

# 39. Redirecionando stderr

Execute:

```bash
ls arquivo-inexistente
```

Agora:

```bash
ls arquivo-inexistente 2> error.log
```

Observe que o erro não apareceu mais no terminal.

Confira:

```bash
cat error.log
```

A mensagem de erro foi enviada para o arquivo.

O número:

```text
2
```

representa `stderr`.

---

# 40. stdout e stderr são independentes

Execute:

```bash
ls config arquivo-inexistente
```

O comando produzirá:

* uma saída válida;
* uma mensagem de erro.

Agora:

```bash
ls config arquivo-inexistente > output.log 2> error.log
```

Confira:

```bash
cat output.log
```

Depois:

```bash
cat error.log
```

Os dois fluxos foram tratados separadamente.

---

# 41. Redirecionando stdin

O operador:

```text
<
```

permite utilizar um arquivo como entrada.

Execute:

```bash
wc -l < logs/application.log
```

Compare com:

```bash
wc -l logs/application.log
```

Os dois comandos contam linhas, mas a origem da entrada é tratada de forma diferente.

---

# 42. Pipes

O operador:

```text
|
```

é chamado de **pipe**.

Ele conecta:

```text
stdout do programa A
           ↓
           |
           ↓
stdin do programa B
```

Isso permite combinar programas.

---

# 43. Primeiro pipeline

Execute:

```bash
cat logs/application.log | grep "ERROR"
```

O fluxo é:

```text
application.log
       ↓
      cat
       ↓ stdout
       |
       ↓ stdin
      grep
       ↓
    terminal
```

Nesse exemplo específico, o `cat` não é necessário.

Também podemos escrever:

```bash
grep "ERROR" logs/application.log
```

Mas a primeira forma ajuda a visualizar como os pipes funcionam.

---

# 44. Contando erros

Podemos combinar `grep` e `wc`.

```bash
grep "ERROR" logs/application.log | wc -l
```

O primeiro comando encontra as linhas de erro.

O segundo conta quantas linhas recebeu.

O resultado é a quantidade de erros registrados.

---

# 45. Pipeline com várias etapas

Execute:

```bash
grep "ERROR" logs/application.log | cut -d' ' -f3
```

O campo `3` representa o nome do componente:

```text
api
worker
api
api
```

Agora:

```bash
grep "ERROR" logs/application.log | cut -d' ' -f3 | sort
```

Resultado semelhante:

```text
api
api
api
worker
```

---

# 46. `sort`

`sort` ordena linhas.

Experimente:

```bash
cut -d' ' -f2 logs/application.log | sort
```

Isso extrai os níveis dos logs e os ordena.

Resultado semelhante:

```text
ERROR
ERROR
ERROR
ERROR
INFO
INFO
INFO
INFO
INFO
WARN
WARN
```

---

# 47. `uniq`

`uniq` identifica linhas repetidas adjacentes.

Por isso, normalmente aparece combinado com `sort`.

Execute:

```bash
cut -d' ' -f2 logs/application.log | sort | uniq
```

Resultado:

```text
ERROR
INFO
WARN
```

---

# 48. Contando ocorrências com `uniq`

Utilize:

```bash
cut -d' ' -f2 logs/application.log | sort | uniq -c
```

Resultado semelhante:

```text
4 ERROR
5 INFO
2 WARN
```

O número exato pode variar caso você tenha adicionado novas linhas ao arquivo durante os exercícios anteriores.

---

# 49. Entendendo o pipeline

O comando:

```bash
cut -d' ' -f2 logs/application.log | sort | uniq -c
```

pode ser interpretado assim:

```text
logs/application.log
        ↓
       cut
        ↓
Extrai o nível do log
        ↓
       sort
        ↓
Agrupa valores iguais
        ↓
     uniq -c
        ↓
Conta ocorrências
```

Pequenos programas foram combinados para responder uma pergunta maior:

> Quantos eventos existem de cada nível?

---

# 50. Descobrindo quais componentes estão apresentando erros

Execute:

```bash
grep "ERROR" logs/application.log
```

Depois:

```bash
grep "ERROR" logs/application.log | cut -d' ' -f3
```

Depois:

```bash
grep "ERROR" logs/application.log | cut -d' ' -f3 | sort
```

Finalmente:

```bash
grep "ERROR" logs/application.log | cut -d' ' -f3 | sort | uniq -c
```

Agora conseguimos responder:

> Qual componente está produzindo mais erros?

Esse é um exemplo simples do tipo de raciocínio utilizado durante troubleshooting.

---

# 51. `tee`

Normalmente, quando utilizamos um pipe, a saída segue para o próximo programa.

O comando `tee` permite:

* mostrar a saída;
* salvar uma cópia em arquivo;
* continuar o pipeline.

Execute:

```bash
grep "ERROR" logs/application.log | tee logs/errors.log
```

Confira:

```bash
cat logs/errors.log
```

---

## Continuando o pipeline

```bash
grep "ERROR" logs/application.log | tee logs/errors.log | wc -l
```

O fluxo agora é:

```text
application.log
      ↓
    grep
      ↓
     tee ──────▶ logs/errors.log
      ↓
     wc
      ↓
 quantidade
```

Esse padrão é muito útil em automações.

---

# 52. Combinando `head`, `tail` e pipes

Primeiras cinco linhas:

```bash
head -n 5 logs/application.log
```

Últimas cinco:

```bash
tail -n 5 logs/application.log
```

Entre as últimas cinco linhas do arquivo, mostrar apenas as que contêm `ERROR`:

```bash
tail -n 5 logs/application.log | grep "ERROR"
```

Se a intenção for obter os **últimos cinco eventos de erro**, a ordem deve ser invertida:

```bash
grep "ERROR" logs/application.log | tail -n 5
```

Em ambientes reais, operações desse tipo são comuns durante análise de logs.

---

# 53. Trabalhando com CSV

Visualize:

```bash
cat data/users.csv
```

Resultado:

```text
id,name,team
1,Ana,backend
2,Carlos,frontend
3,Marina,backend
4,Pedro,devops
5,Julia,devops
```

---

## Extraindo uma coluna

Utilize:

```bash
cut -d',' -f3 data/users.csv
```

Resultado:

```text
team
backend
frontend
backend
devops
devops
```

Onde:

```text
-d','  → delimitador é vírgula

-f3    → terceiro campo
```

---

## Removendo o cabeçalho

Podemos utilizar:

```bash
tail -n +2 data/users.csv
```

Depois:

```bash
tail -n +2 data/users.csv | cut -d',' -f3
```

---

## Contando usuários por equipe

Agora combine:

```bash
tail -n +2 data/users.csv | cut -d',' -f3 | sort | uniq -c
```

Resultado:

```text
2 backend
2 devops
1 frontend
```

Observe como comandos simples começaram a funcionar como uma pequena ferramenta de análise de dados.

---

# 54. Laboratório — Investigando um incidente

Considere o seguinte cenário:

> A equipe recebeu um alerta informando que a aplicação está apresentando erros. Você recebeu o arquivo `logs/application.log` e precisa realizar uma investigação inicial.

Não tente resolver tudo em uma única linha imediatamente.

Construa a análise progressivamente.

---

## Etapa 1 — Quantas linhas existem no log?

Utilize o comando:

```text
wc
```

---

## Etapa 2 — Quais eventos são erros?

Utilize o comando:

```text
grep
```

---

## Etapa 3 — Quantos erros existem?

Combine:

```text
grep
+
wc
```

---

## Etapa 4 — Quais componentes geraram erros?

Combine:

```text
grep
+
cut
```

---

## Etapa 5 — Quantos erros foram produzidos por cada componente?

Combine:

```text
grep
+
cut
+
sort
+
uniq
```

---

## Etapa 6 — Salve os erros encontrados

Salve os eventos em:

```text
logs/errors.log
```

Sem perder a visualização no terminal.

Utilize:

```text
tee
```

---

# 55. Solução do laboratório

<details>

<summary>Mostrar solução</summary>

### Número de linhas

```bash
wc -l logs/application.log
```

### Visualizar erros

```bash
grep "ERROR" logs/application.log
```

### Quantidade de erros

```bash
grep "ERROR" logs/application.log | wc -l
```

### Componentes com erro

```bash
grep "ERROR" logs/application.log | cut -d' ' -f3
```

### Quantidade por componente

```bash
grep "ERROR" logs/application.log \
  | cut -d' ' -f3 \
  | sort \
  | uniq -c
```

Também pode ser escrito em uma linha:

```bash
grep "ERROR" logs/application.log | cut -d' ' -f3 | sort | uniq -c
```

### Salvar os erros

```bash
grep "ERROR" logs/application.log | tee logs/errors.log
```

</details>

---

# 56. Desafio

Utilizando apenas os comandos apresentados nesta prática, descubra:

1. quantos eventos possuem status HTTP `500`;
2. quantos eventos possuem nível `WARN`;
3. quais níveis de log existem;
4. quantas ocorrências existem de cada nível;
5. quais componentes aparecem no arquivo;
6. quantas vezes cada componente aparece;
7. quais linhas possuem `/health`;
8. quais eventos não possuem nível `INFO`;
9. quantas linhas existem no arquivo `data/users.csv`;
10. quantos usuários existem em cada equipe.

---

# 57. Solução do desafio

<details>

<summary>Mostrar solução</summary>

### 1. Eventos HTTP 500

```bash
grep " 500 " logs/application.log | wc -l
```

### 2. Eventos WARN

```bash
grep "WARN" logs/application.log | wc -l
```

### 3. Níveis existentes

```bash
cut -d' ' -f2 logs/application.log | sort | uniq
```

### 4. Quantidade por nível

```bash
cut -d' ' -f2 logs/application.log | sort | uniq -c
```

### 5. Componentes existentes

```bash
cut -d' ' -f3 logs/application.log | sort | uniq
```

### 6. Quantidade por componente

```bash
cut -d' ' -f3 logs/application.log | sort | uniq -c
```

### 7. Eventos de health check

```bash
grep "/health" logs/application.log
```

### 8. Eventos diferentes de INFO

```bash
grep -v "INFO" logs/application.log
```

### 9. Linhas no CSV

```bash
wc -l data/users.csv
```

### 10. Usuários por equipe

```bash
tail -n +2 data/users.csv | cut -d',' -f3 | sort | uniq -c
```

</details>

---

# 58. Um exemplo próximo da rotina DevOps

Imagine que uma aplicação produza milhares de linhas de log. Continuando com o arquivo utilizado nesta prática, uma investigação inicial poderia começar com:

```bash
tail -n 1000 logs/application.log
```

Depois:

```bash
tail -n 1000 logs/application.log | grep "ERROR"
```

Depois:

```bash
tail -n 1000 logs/application.log | grep "ERROR" | wc -l
```

Talvez também seja necessário identificar quais componentes apresentam problemas:

```bash
tail -n 1000 logs/application.log \
  | grep "ERROR" \
  | cut -d' ' -f3 \
  | sort \
  | uniq -c
```

A ideia principal não é decorar essa sequência.

O importante é compreender que:

```text
cada comando resolve uma pequena parte do problema
```

e:

```text
pipes permitem combinar essas pequenas operações
```

---

# 59. Comandos utilizados nesta prática

| Comando    | Finalidade                                   |
| ---------- | -------------------------------------------- |
| `whoami`   | Mostrar usuário atual                        |
| `hostname` | Mostrar nome da máquina                      |
| `uname`    | Informações sobre kernel e sistema           |
| `ps`       | Exibir informações sobre processos           |
| `pwd`      | Mostrar diretório atual                      |
| `ls`       | Listar arquivos e diretórios                 |
| `cd`       | Mudar de diretório                           |
| `mkdir`    | Criar diretório                              |
| `touch`    | Criar arquivo vazio ou atualizar timestamp   |
| `cp`       | Copiar arquivos                              |
| `mv`       | Mover ou renomear                            |
| `rm`       | Remover arquivos                             |
| `rmdir`    | Remover diretórios vazios                    |
| `cat`      | Exibir conteúdo                              |
| `less`     | Navegar por conteúdo                         |
| `head`     | Mostrar início de um arquivo                 |
| `tail`     | Mostrar final de um arquivo                  |
| `file`     | Identificar tipo de arquivo                  |
| `stat`     | Exibir metadados                             |
| `ln`       | Criar links                                  |
| `find`     | Localizar arquivos                           |
| `grep`     | Pesquisar padrões                            |
| `wc`       | Contar linhas, palavras, bytes ou caracteres |
| `cut`      | Extrair campos                               |
| `sort`     | Ordenar linhas                               |
| `uniq`     | Agrupar/remover linhas duplicadas adjacentes |
| `tee`      | Mostrar e copiar um fluxo                    |
| `man`      | Consultar documentação                       |

---

# 60. Operadores importantes

| Operador | Função |
| --- | --- |
| `.` | Diretório atual |
| `..` | Diretório pai |
| `~` | Diretório home |
| `*` | Corresponde a zero ou mais caracteres em padrões do shell |
| `>` | Redireciona stdout sobrescrevendo |
| `>>` | Redireciona stdout acrescentando |
| `<` | Redireciona stdin |
| `2>` | Redireciona stderr |
| <code>&#124;</code> | Conecta stdout ao stdin de outro programa |

---

# 61. Boas práticas no terminal

Antes de executar comandos destrutivos, confirme onde você está:

```bash
pwd
```

Confira os arquivos:

```bash
ls
```

Tenha cuidado principalmente com `rm` e `rm -r`.

Em ambientes de produção, valide sempre:

```text
servidor
↓
diretório
↓
comando
↓
argumentos
```

antes de pressionar `Enter`.

---

# 62. Limpando o laboratório

Ao final da prática, volte para seu diretório home:

```bash
cd ~
```

Confirme:

```bash
pwd
```

Liste o laboratório:

```bash
ls ~/devops-linux-lab
```

Caso queira removê-lo:

```bash
rm -r ~/devops-linux-lab
```

> Execute a remoção apenas depois de confirmar que o caminho está correto.

---

# Checklist

Ao concluir a prática, verifique se você consegue realizar as seguintes tarefas sem consultar diretamente a solução:

* descobrir usuário, hostname, distribuição e versão do kernel;
* identificar seu diretório atual;
* navegar utilizando caminhos absolutos e relativos;
* compreender `.`, `..` e `~`;
* criar arquivos e diretórios;
* copiar, mover e remover arquivos;
* visualizar início e final de arquivos;
* acompanhar um log com `tail -f`;
* localizar arquivos com `find`;
* pesquisar conteúdo utilizando `grep`;
* diferenciar `stdin`, `stdout` e `stderr`;
* utilizar `>`, `>>`, `<` e `2>`;
* construir pipelines utilizando `|`;
* combinar `grep`, `cut`, `sort`, `uniq` e `wc`;
* utilizar `tee` para visualizar e salvar uma saída.

---

# Próxima prática

Na próxima aula serão explorados:

**Usuários e grupos**

**Propriedade de arquivos**

**Permissões**

**`chmod`**

**`chown`**

**`sudo`**

Esses conceitos serão utilizados para compreender como Linux controla **quem pode acessar, modificar ou executar recursos do sistema**.

---

# Referências

**NEMETH, E.; SNYDER, G.; HEIN, T. R.; WHALEY, B.; MACKIN, D.**
*UNIX and Linux System Administration Handbook*. 5. ed. Pearson, 2018.

Capítulos utilizados:

* **Cap. 1 — Where to Start**
* **Cap. 5 — The Filesystem**
* **Cap. 7 — Scripting and the Shell**