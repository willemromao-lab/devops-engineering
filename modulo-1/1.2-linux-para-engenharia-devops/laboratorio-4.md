# 1.2 — Linux para Engenharia DevOps

## Laboratório 4 — Bash, Variáveis de Ambiente, Argumentos, Condicionais, Loops e Automação com Shell Script

Este material complementa a aula teórica **Linux para Engenharia DevOps — Parte 4**.

O objetivo desta prática é transformar comandos isolados em **scripts Bash confiáveis**, terminando com uma pequena automação de validação e implantação local de arquivos. A implantação é uma **simulação didática**: não utiliza servidores externos, credenciais, serviços de produção ou ferramentas de CI/CD.

> Os exemplos foram pensados para distribuições Linux com Bash, como Ubuntu, Debian, Fedora e Rocky Linux. **Não é necessário ter systemd como PID 1 nem utilizar `sudo`.** O laboratório também pode ser executado em um container ou sandbox Linux com as ferramentas indicadas.

> Execute esta prática em uma VM, container ou ambiente descartável. Todos os arquivos criados deverão permanecer em `~/lab-devops-bash-04`. Não substitua os caminhos utilizados por diretórios reais de aplicações ou sistemas.

## Pré-requisitos e validação do ambiente

Confirme que o Bash está disponível:

```bash
bash --version
```

Confira as principais ferramentas:

```bash
command -v bash
command -v mkdir
command -v cp
command -v cmp
command -v cat
command -v grep
command -v sed
command -v find
command -v chmod
command -v date
command -v dirname
command -v sort
```

O comando `command -v` deve apresentar o caminho dos programas disponíveis (ou indicar, sem saída e com código diferente de zero, que algum deles não foi encontrado).

> Em distribuições Linux convencionais, essas ferramentas normalmente já estão instaladas. Para este laboratório, utilize **Bash 4 ou superior**. O ShellCheck será apresentado como uma verificação **opcional**; sua instalação não é um pré-requisito.

**Importante:** os comandos foram organizados para execução no **mesmo terminal**, pois utilizaremos a variável `LAB_DIR`. Se abrir outro terminal, defina-a novamente conforme a seção 1.

> **Antes de começar:** utilize um ambiente no qual `~/lab-devops-bash-04` ainda não exista. Algumas etapas recriam arquivos de exemplo e podem sobrescrever versões anteriores do próprio laboratório. Se você já realizou parte da prática, revise os arquivos existentes antes de repeti-la.

---

## Objetivos

Ao final deste laboratório, você deverá ser capaz de:

- compreender o funcionamento de um script Bash e o papel do *shebang*;
- diferenciar executar um script de carregar comandos com `source`;
- utilizar variáveis, expansões e aspas de maneira segura;
- diferenciar variáveis da shell, variáveis de ambiente e argumentos;
- usar substituição de comandos e expressões aritméticas;
- distinguir `stdout`, `stderr` e códigos de saída;
- tomar decisões com `if` e `case`;
- percorrer valores e linhas com `for` e `while`;
- organizar instruções em funções;
- validar entradas e tratar erros em scripts;
- implementar um modo de pré-visualização (*dry-run*);
- criar uma operação local repetível, com comportamento idempotente para arquivos inalterados;
- encadear validação, preparação e verificação final como uma pequena pipeline;
- investigar falhas e verificar o **resultado**, não apenas a execução do comando.

# 1. Preparando um ambiente de trabalho isolado

Crie uma variável com o diretório que será utilizado durante toda a prática:

```bash
export LAB_DIR="$HOME/lab-devops-bash-04"
```

Confirme o valor:

```bash
printf '%s\n' "$LAB_DIR"
```

Agora crie a estrutura inicial:

```bash
mkdir -p "$LAB_DIR/app" \
         "$LAB_DIR/scripts" \
         "$LAB_DIR/inputs" \
         "$LAB_DIR/logs" \
         "$LAB_DIR/deployments"
```

Consulte os diretórios criados:

```bash
find "$LAB_DIR" -maxdepth 2 -type d | sort
```

A estrutura deverá se parecer com:

```text
lab-devops-bash-04/
├── app/
├── deployments/
├── inputs/
├── logs/
└── scripts/
```

> Usaremos sempre caminhos entre aspas. Isso evita problemas quando nomes de arquivos ou diretórios contêm espaços. **Não execute `rm -rf` nem comandos administrativos para preparar este laboratório.**

---

# 2. Criando os arquivos da aplicação de demonstração

Crie o conteúdo da aplicação:

```bash
printf 'Aplicação de demonstração - versão 1\n' > "$LAB_DIR/app/app.txt"
printf 'healthy\n' > "$LAB_DIR/app/health.txt"
printf 'Arquivo cujo nome contém espaços.\n' > "$LAB_DIR/app/guia de operação.txt"
```

Confira os arquivos:

```bash
find "$LAB_DIR/app" -maxdepth 1 -type f | sort
```

Veja o conteúdo dos dois arquivos que representam nossa aplicação:

```bash
cat "$LAB_DIR/app/app.txt"
cat "$LAB_DIR/app/health.txt"
```

Saída esperada:

```text
Aplicação de demonstração - versão 1
healthy
```

Durante o laboratório, consideraremos que:

- `app.txt` representa o artefato principal que será implantado;
- `health.txt` representa uma condição funcional simplificada da aplicação;
- `deployments/` representa o local onde as versões serão publicadas, **apenas dentro do laboratório**.

> Não confunda este arquivo `health.txt` com um health check real de uma aplicação HTTP. Aqui estamos apenas simulando uma validação de conteúdo, sem servidor ou requisições de rede.

---

# 3. Investigando o Bash em execução

Exiba a versão do interpretador utilizado pelo terminal atual:

```bash
printf 'Versão do Bash: %s\n' "$BASH_VERSION"
```

Verifique o PID da shell:

```bash
printf 'PID da shell: %s\n' "$$"
```

Agora peça ao Bash que execute um pequeno comando em uma nova instância:

```bash
bash -c 'printf "PID da nova shell: %s\n" "$BASHPID"'
```

Em geral, os dois PIDs serão diferentes. Isso acontece porque `bash -c` inicia outro interpretador para executar a instrução fornecida.

**Questões**

1. O Bash é o próprio kernel Linux?
2. Qual a diferença entre um comando digitado interativamente e o mesmo comando dentro de um script?
3. Um script pode iniciar outros processos e utilizar comandos externos?

---

# 4. Criando o primeiro Shell Script

Crie o arquivo:

```bash
cat > "$LAB_DIR/scripts/ola.sh" <<'EOF'
#!/usr/bin/env bash

printf 'Olá! Este é meu primeiro Shell Script.\n'
printf 'Diretório atual: %s\n' "$PWD"
EOF
```

Exiba o conteúdo:

```bash
cat "$LAB_DIR/scripts/ola.sh"
```

Execute o arquivo fornecendo-o diretamente ao Bash:

```bash
bash "$LAB_DIR/scripts/ola.sh"
```

O interpretador executará as instruções sequencialmente.

Observe a primeira linha:

```bash
#!/usr/bin/env bash
```

Ela é chamada de *shebang* e indica como executar o arquivo quando ele é iniciado diretamente. Neste exemplo, `env` localiza `bash` usando o `PATH` do processo.

> A extensão `.sh` é uma convenção; não é ela que torna um arquivo executável no Linux.

---

# 5. Execução direta e permissões

Confira as permissões atuais:

```bash
ls -l "$LAB_DIR/scripts/ola.sh"
```

Agora permita a execução direta:

```bash
chmod u+x "$LAB_DIR/scripts/ola.sh"
```

Execute:

```bash
"$LAB_DIR/scripts/ola.sh"
```

Compare os dois modos:

```text
bash arquivo.sh  → o Bash abre e interpreta o arquivo
./arquivo.sh     → o sistema tenta executar o arquivo, usando o shebang
```

> No nosso caso, o caminho completo `"$LAB_DIR/scripts/ola.sh"` funciona como execução direta. Em um diretório de trabalho, também seria possível usar `./ola.sh`.

**Questões**

1. Por que o arquivo funcionou com `bash arquivo.sh` antes de receber permissão de execução?
2. O que mudaria se o *shebang* especificasse outro interpretador?

---

# 6. Executar um script não é o mesmo que usar `source`

Crie um segundo arquivo:

```bash
cat > "$LAB_DIR/scripts/contexto.sh" <<'EOF'
#!/usr/bin/env bash
LAB_MARCA="configurada-pelo-arquivo"
EOF
```

Remova um eventual valor anterior da variável na shell atual:

```bash
unset LAB_MARCA
```

Execute o conteúdo em uma nova instância do Bash:

```bash
bash "$LAB_DIR/scripts/contexto.sh"
printf 'Na shell atual: %s\n' "${LAB_MARCA:-ausente}"
```

Resultado esperado:

```text
Na shell atual: ausente
```

Agora carregue o arquivo **na shell atual**:

```bash
source "$LAB_DIR/scripts/contexto.sh"
printf 'Na shell atual: %s\n' "$LAB_MARCA"
```

Resultado esperado:

```text
Na shell atual: configurada-pelo-arquivo
```

O `source` é útil para carregar configurações, mas também executa qualquer comando contido no arquivo. Portanto, **não utilize `source` em arquivos desconhecidos ou não confiáveis**.

Limpe a variável utilizada no teste:

```bash
unset LAB_MARCA
```

---

# 7. Variáveis e expansão de valores

Defina algumas variáveis:

```bash
APP_NAME="catalogo"
APP_VERSION="1.0"
```

Exiba seus valores:

```bash
printf 'Aplicação: %s\n' "$APP_NAME"
printf 'Versão: %s\n' "$APP_VERSION"
```

Combine uma variável com outros caracteres:

```bash
printf 'Artefato: %s\n' "${APP_NAME}-${APP_VERSION}.txt"
```

Observe a utilização de `${APP_NAME}` e `${APP_VERSION}` para delimitar claramente os nomes das variáveis.

Agora compare:

```bash
printf '%s\n' 'Literal: $APP_NAME'
printf '%s\n' "Expandido: $APP_NAME"
```

Saída esperada:

```text
Literal: $APP_NAME
Expandido: catalogo
```

**Conceito**

- Aspas simples preservam o texto literalmente.
- Aspas duplas permitem expansão de variáveis e, ao mesmo tempo, preservam valores com espaços como um único argumento.

---

# 8. Por que o uso correto de aspas importa?

Consulte o arquivo cujo nome contém espaços:

```bash
cat "$LAB_DIR/app/guia de operação.txt"
```

Em seguida, armazene seu caminho em uma variável:

```bash
GUIA="$LAB_DIR/app/guia de operação.txt"
```

Utilize essa variável corretamente:

```bash
cat "$GUIA"
```

O conteúdo deve ser exibido sem problemas.

> Evite usar variáveis de caminho sem aspas, como em `cat $GUIA`: o Bash pode interpretar palavras separadas por espaços como argumentos diferentes. Esse detalhe causa falhas frequentes em automações.

Veja também um padrão de expansão de arquivos:

```bash
for arquivo in "$LAB_DIR"/app/*.txt; do
    printf '%s\n' "$arquivo"
done
```

Nesse exemplo, **a parte variável do caminho permanece entre aspas** e o padrão `*.txt` fica fora delas para permitir a expansão dos nomes de arquivos.

---

# 9. Variáveis de ambiente e herança entre processos

Defina uma variável de ambiente:

```bash
export APP_ENV="dev"
```

Verifique se um novo processo Bash consegue recebê-la:

```bash
bash -c 'printf "APP_ENV no processo filho: %s\n" "$APP_ENV"'
```

Resultado esperado:

```text
APP_ENV no processo filho: dev
```

Agora substitua o valor **apenas para uma execução**:

```bash
APP_ENV="staging" bash -c 'printf "APP_ENV no filho: %s\n" "$APP_ENV"'
```

Depois, confira a shell atual:

```bash
printf 'APP_ENV na shell atual: %s\n' "$APP_ENV"
```

Resultado esperado:

```text
APP_ENV na shell atual: dev
```

Essa distinção será importante: o mesmo script poderá funcionar em `dev` ou `staging` de acordo com a variável de ambiente recebida.

> Variáveis de ambiente **não são um cofre de segredos**. Não coloque senhas, tokens reais ou credenciais nos exemplos deste laboratório.

---

# 10. Argumentos posicionais e `"$@"`

Crie um script que recebe argumentos:

```bash
cat > "$LAB_DIR/scripts/argumentos.sh" <<'EOF'
#!/usr/bin/env bash

printf 'Nome usado para executar o script: %s\n' "$0"
printf 'Quantidade de argumentos: %s\n' "$#"
printf 'Primeiro argumento: %s\n' "${1:-<ausente>}"
printf 'Segundo argumento: %s\n' "${2:-<ausente>}"

for argumento in "$@"; do
    printf 'Argumento recebido: <%s>\n' "$argumento"
done
EOF
```

Execute com dois argumentos:

```bash
bash "$LAB_DIR/scripts/argumentos.sh" "origem com espaços" "release-01"
```

Confira a saída. O texto `origem com espaços` deve aparecer como **um único argumento**.

Observe também:

- `$0` identifica o nome/caminho usado para iniciar o script;
- `$1`, `$2` etc. representam argumentos posicionais;
- `$#` informa quantos argumentos foram recebidos;
- `"$@"` preserva individualmente os argumentos, inclusive os que contêm espaços.

**Questões**

1. O que acontece quando nenhum argumento é informado?
2. Por que um script reutilizável deve validar os argumentos recebidos?

---

# 11. Substituição de comandos e aritmética

Armazene a saída de um comando:

```bash
DATA_EXECUCAO=$(date +%Y-%m-%d)
printf 'Data da execução: %s\n' "$DATA_EXECUCAO"
```

A construção `$(...)` executa um comando e utiliza sua saída como valor.

Experimente uma operação aritmética:

```bash
CONTADOR=2
CONTADOR=$((CONTADOR + 1))
printf 'Contador: %s\n' "$CONTADOR"
```

Resultado esperado:

```text
Contador: 3
```

> Variáveis comuns da shell armazenam texto. A construção `$((...))` solicita avaliação aritmética para a expressão indicada.

---

# 12. Separando saída normal e mensagens de erro

Crie:

```bash
cat > "$LAB_DIR/scripts/saidas.sh" <<'EOF'
#!/usr/bin/env bash

printf 'Operação concluída.\n'
printf 'Aviso de teste enviado ao stderr.\n' >&2
EOF
```

Execute redirecionando cada saída para um arquivo:

```bash
bash "$LAB_DIR/scripts/saidas.sh" \
    > "$LAB_DIR/logs/saida-normal.log" \
    2> "$LAB_DIR/logs/saida-erro.log"
```

Confira os resultados:

```bash
cat "$LAB_DIR/logs/saida-normal.log"
cat "$LAB_DIR/logs/saida-erro.log"
```

Interprete os descritores:

```text
stdout (1) → saída normal do programa
stderr (2) → mensagens de erro ou diagnóstico
```

O nome `saida-erro.log` é apenas um arquivo escolhido para capturar `stderr`: nem toda mensagem enviada para essa saída representa uma falha.

---

# 13. Códigos de saída: sucesso e falha

Execute um comando que representa sucesso:

```bash
true
printf 'Código de saída: %s\n' "$?"
```

Resultado esperado:

```text
Código de saída: 0
```

Agora execute um comando que representa falha:

```bash
false
printf 'Código de saída: %s\n' "$?"
```

Resultado esperado:

```text
Código de saída: 1
```

`$?` apresenta o resultado do **último comando**, então consulte-o imediatamente após a operação desejada.

Teste o encadeamento condicional:

```bash
true && printf 'Executado porque houve sucesso.\n'
false || printf 'Executado porque houve falha.\n'
```

Em automações, o código de saída determina se uma etapa pode ser considerada bem-sucedida. O fato de um comando imprimir mensagens não significa necessariamente que ele tenha retornado sucesso.

---

# 14. Utilizando `if` para verificar arquivos

Verifique se o arquivo principal existe:

```bash
if [[ -f "$LAB_DIR/app/app.txt" ]]; then
    printf '[OK] Artefato encontrado.\n'
else
    printf '[ERRO] Artefato ausente.\n' >&2
fi
```

Agora verifique se o arquivo está vazio:

```bash
if [[ -s "$LAB_DIR/app/app.txt" ]]; then
    printf '[OK] Artefato não está vazio.\n'
else
    printf '[ERRO] Artefato está vazio ou ausente.\n' >&2
fi
```

Nesse contexto:

- `-f` verifica se existe um arquivo regular;
- `-d` verifica se existe um diretório;
- `-s` verifica se o arquivo existe e possui tamanho maior que zero;
- `!` inverte o resultado de uma condição.

> `[[ ... ]]` é uma construção do Bash. Para scripts que precisam executar em uma shell estritamente POSIX, a compatibilidade e a sintaxe devem ser avaliadas separadamente.

---

# 15. Selecionando ambientes com `case`

O `case` é útil quando existem alternativas conhecidas.

```bash
case "$APP_ENV" in
    dev)
        printf 'Ambiente de desenvolvimento selecionado.\n'
        ;;
    staging)
        printf 'Ambiente de homologação selecionado.\n'
        ;;
    *)
        printf 'Ambiente não suportado: %s\n' "$APP_ENV" >&2
        ;;
esac
```

Experimente modificar `APP_ENV` **somente para um comando**:

```bash
APP_ENV=staging bash -c 'case "$APP_ENV" in dev) echo desenvolvimento ;; staging) echo homologação ;; *) echo inválido ;; esac'
```

Resultado esperado:

```text
homologação
```

No script final, valores fora das opções permitidas provocarão uma falha explícita em vez de serem tratados silenciosamente.

---

# 16. Repetindo operações com `for`

Precisamos validar dois arquivos fundamentais de nossa aplicação.

```bash
for arquivo in app.txt health.txt; do
    if [[ -s "$LAB_DIR/app/$arquivo" ]]; then
        printf '[OK] %s\n' "$arquivo"
    else
        printf '[ERRO] %s\n' "$arquivo" >&2
    fi
done
```

Resultado esperado:

```text
[OK] app.txt
[OK] health.txt
```

Nesse exemplo, a lista de arquivos é conhecida. Cada iteração utiliza um nome de arquivo diferente, mantendo a mesma lógica de verificação.

**Questão**

Como você alteraria o `for` para verificar também um arquivo chamado `config.txt`?

---

# 17. Processando linhas de um arquivo com `while`

Crie uma lista de componentes:

```bash
cat > "$LAB_DIR/inputs/componentes.txt" <<'EOF'
api
worker

gateway
EOF
```

Agora percorra o arquivo linha por linha:

```bash
while IFS= read -r componente || [[ -n "$componente" ]]; do
    [[ -z "$componente" ]] && continue
    printf 'Componente identificado: %s\n' "$componente"
done < "$LAB_DIR/inputs/componentes.txt"
```

Resultado esperado:

```text
Componente identificado: api
Componente identificado: worker
Componente identificado: gateway
```

Observe:

- `IFS=` evita que o `read` descarte automaticamente espaços iniciais e finais;
- `-r` evita interpretar barras invertidas como escapes;
- `continue` ignora a linha em branco;
- a condição adicional trata corretamente uma última linha sem quebra de linha final.

> Um `while` também pode consultar repetidamente o estado de um serviço. Nesse caso, defina um tempo máximo de espera para não criar um loop infinito.

---

# 18. Organizando comportamentos com funções

Crie:

```bash
cat > "$LAB_DIR/scripts/funcoes.sh" <<'EOF'
#!/usr/bin/env bash

registrar() {
    printf '[INFO] %s\n' "$1"
}

verificar_arquivo() {
    local caminho=$1

    if [[ -s "$caminho" ]]; then
        registrar "Arquivo válido: $caminho"
        return 0
    fi

    printf '[ERRO] Arquivo ausente ou vazio: %s\n' "$caminho" >&2
    return 1
}

verificar_arquivo "$1"
EOF
```

Execute fornecendo um arquivo válido:

```bash
bash "$LAB_DIR/scripts/funcoes.sh" "$LAB_DIR/app/app.txt"
```

> Uma função pode receber seus próprios parâmetros posicionais. No exemplo, o `local` mantém a variável `caminho` dentro do escopo da função.

Execute com um caminho inexistente, tratando a falha no terminal:

```bash
if bash "$LAB_DIR/scripts/funcoes.sh" "$LAB_DIR/app/nao-existe.txt"; then
    printf 'Resultado inesperado: sucesso.\n'
else
    printf 'Falha detectada corretamente.\n'
fi
```

O script deve escrever uma mensagem em `stderr` e retornar um código diferente de zero.

---

# 19. Planejando uma automação segura

Agora vamos construir uma operação reutilizável para preparar uma **release local**.

O fluxo desejado é:

```text
Receber origem e identificador da release
                ↓
Validar argumentos e ambiente
                ↓
Verificar app.txt e health.txt
                ↓
Pré-visualizar ou copiar os arquivos
                ↓
Validar o destino
                ↓
Informar sucesso ou falha
```

Antes de copiar, experimente apenas **mostrar** o que seria feito:

```bash
for arquivo in app.txt health.txt; do
    printf '[DRY-RUN] Copiaria %s\n' "$LAB_DIR/app/$arquivo"
done
```

O *dry-run* é uma pré-visualização. Um script que oferece esse modo não deve alterar o destino durante a simulação.

> Este exercício manipulará somente arquivos pertencentes ao laboratório. Não será executado nenhum deployment real em servidores, containers ou clusters.

---

# 20. Construindo o script de validação `preflight.sh`

Vamos começar pelo *preflight*: uma verificação dos requisitos antes de qualquer implantação.

Crie o script completo:

```bash
cat > "$LAB_DIR/scripts/preflight.sh" <<'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

uso() {
    printf 'Uso: %s <diretorio-da-aplicacao>\n' "$0"
}

falhar() {
    printf '[ERRO] %s\n' "$1" >&2
    exit "${2:-1}"
}

if (( $# != 1 )); then
    uso >&2
    exit 2
fi

origem=$1
ambiente=${APP_ENV:-dev}

case "$ambiente" in
    dev|staging) ;;
    *) falhar "APP_ENV inválido: $ambiente" 2 ;;
esac

[[ -d "$origem" ]] || falhar "Diretório ausente: $origem"

for arquivo in app.txt health.txt; do
    [[ -s "$origem/$arquivo" ]] || falhar "Arquivo ausente ou vazio: $origem/$arquivo"
done

saude=$(<"$origem/health.txt")
[[ "$saude" == "healthy" ]] || falhar "Estado de saúde inválido: $saude"

printf '[OK] Preflight aprovado: ambiente=%s origem=%s\n' "$ambiente" "$origem"
EOF
```

O script utiliza conceitos vistos anteriormente:

- `set -Eeuo pipefail` habilita verificações úteis de execução, variáveis e pipelines; **não substitui tratamento explícito de erros**;
- `uso` e `falhar` organizam mensagens e códigos de saída;
- os argumentos são verificados antes do uso;
- `case` restringe os ambientes aceitos;
- `for` verifica os arquivos necessários;
- o conteúdo de `health.txt` também é validado, e não apenas sua existência.

**Importante:** o `set -e` possui exceções e contextos específicos em Bash. Por isso, as falhas que queremos explicar ao usuário são tratadas diretamente, com testes e mensagens apropriadas.

---

# 21. Validando a sintaxe antes de executar

Utilize o próprio Bash para analisar a sintaxe:

```bash
bash -n "$LAB_DIR/scripts/preflight.sh"
```

Se o arquivo estiver sintaticamente correto, o comando normalmente não exibirá nenhuma mensagem e retornará `0`.

Agora torne o script executável:

```bash
chmod u+x "$LAB_DIR/scripts/preflight.sh"
```

Execute a verificação:

```bash
"$LAB_DIR/scripts/preflight.sh" "$LAB_DIR/app"
```

Resultado esperado:

```text
[OK] Preflight aprovado: ambiente=dev origem=.../lab-devops-bash-04/app
```

O caminho completo dependerá do diretório pessoal utilizado.

> Uma verificação de sintaxe **não comprova** que a lógica do script está correta. Precisamos testar entradas válidas e inválidas.

---

# 22. Testando entradas incorretas

Primeiro, execute o script **sem argumentos**:

```bash
if "$LAB_DIR/scripts/preflight.sh"; then
    printf 'Resultado inesperado: sucesso.\n'
else
    printf 'Argumentos inválidos foram rejeitados.\n'
fi
```

O script deve apresentar a forma de uso e retornar código `2`.

Agora informe um ambiente não permitido:

```bash
if APP_ENV=producao "$LAB_DIR/scripts/preflight.sh" "$LAB_DIR/app"; then
    printf 'Resultado inesperado: sucesso.\n'
else
    printf 'Ambiente inválido foi rejeitado.\n'
fi
```

Teste um diretório inexistente:

```bash
if "$LAB_DIR/scripts/preflight.sh" "$LAB_DIR/app-inexistente"; then
    printf 'Resultado inesperado: sucesso.\n'
else
    printf 'Diretório inexistente foi detectado.\n'
fi
```

As falhas desses testes são **esperadas**. O importante é que o script informe claramente o problema e não retorne sucesso.

---

# 23. Simulando uma aplicação com falha funcional

Crie uma segunda versão de teste com o arquivo de saúde incorreto:

```bash
mkdir -p "$LAB_DIR/app-com-falha"
cp "$LAB_DIR/app/app.txt" "$LAB_DIR/app-com-falha/app.txt"
printf 'unhealthy\n' > "$LAB_DIR/app-com-falha/health.txt"
```

Verifique:

```bash
cat "$LAB_DIR/app-com-falha/health.txt"
```

Resultado esperado:

```text
unhealthy
```

Agora execute o *preflight*:

```bash
if "$LAB_DIR/scripts/preflight.sh" "$LAB_DIR/app-com-falha"; then
    printf 'Resultado inesperado: sucesso.\n'
else
    printf 'Falha funcional detectada corretamente.\n'
fi
```

O script deverá rejeitar o conteúdo `unhealthy`.

**Reflexão**

O arquivo existe e não está vazio. Por que isso não é suficiente para declarar a aplicação saudável?

> Assim como no Laboratório 3, **existir e estar em execução não é o mesmo que funcionar corretamente**. Neste exercício, a validação do conteúdo é nossa evidência simplificada de saúde.

---

# 24. Construindo a automação `deploy.sh`

Agora criaremos o script principal. Ele receberá a **origem**, o **identificador da release** e, opcionalmente, `--dry-run`.

A publicação ocorrerá apenas em:

```text
~/lab-devops-bash-04/deployments/<ambiente>/<release>/
```

Crie o arquivo:

```bash
cat > "$LAB_DIR/scripts/deploy.sh" <<'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

script_dir=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)
lab_dir=$(cd -- "$script_dir/.." && pwd -P)

uso() {
    printf 'Uso: %s <origem> <release> [--dry-run]\n' "$0"
}

falhar() {
    printf '[ERRO] %s\n' "$1" >&2
    exit "${2:-1}"
}

if (( $# < 2 || $# > 3 )); then
    uso >&2
    exit 2
fi

origem=$1
release=$2
modo=${3:-}
ambiente=${APP_ENV:-dev}

case "$modo" in
    ''|--dry-run) ;;
    *) uso >&2; falhar "Opção desconhecida: $modo" 2 ;;
esac

case "$ambiente" in
    dev|staging) ;;
    *) falhar "APP_ENV inválido: $ambiente" 2 ;;
esac

[[ "$release" =~ ^[a-zA-Z0-9][a-zA-Z0-9._-]*$ ]] || \
    falhar "Identificador de release inválido: $release" 2

"$script_dir/preflight.sh" "$origem"

destino_raiz="$lab_dir/deployments"
destino_ambiente="$destino_raiz/$ambiente"
destino="$destino_ambiente/$release"

# Evita seguir links simbólicos nos diretórios usados para a publicação.
for caminho in "$destino_raiz" "$destino_ambiente" "$destino"; do
    [[ ! -L "$caminho" ]] || falhar "Destino não permitido (link simbólico): $caminho"
done

[[ ! -e "$destino" || -d "$destino" ]] || \
    falhar "Destino existe e não é um diretório: $destino"

if [[ "$modo" == "--dry-run" ]]; then
    printf '[DRY-RUN] Publicaria %s em %s\n' "$origem" "$destino"
    exit 0
fi

mkdir -p -- "$destino"

for arquivo in app.txt health.txt; do
    origem_arquivo="$origem/$arquivo"
    destino_arquivo="$destino/$arquivo"

    [[ ! -L "$destino_arquivo" ]] || \
        falhar "Arquivo de destino não permitido (link simbólico): $destino_arquivo"

    if [[ -f "$destino_arquivo" ]] && cmp -s -- "$origem_arquivo" "$destino_arquivo"; then
        printf '[SKIP] Sem alterações: %s\n' "$arquivo"
    else
        cp -- "$origem_arquivo" "$destino_arquivo"
        printf '[COPY] Arquivo publicado: %s\n' "$arquivo"
    fi
done

"$script_dir/preflight.sh" "$destino"
printf '[OK] Release publicada: ambiente=%s release=%s\n' "$ambiente" "$release"
EOF
```

Repare que o script:

1. encontra o diretório do próprio script e o diretório raiz do laboratório;
2. limita os ambientes e o formato do identificador da release;
3. valida a origem **antes** de qualquer cópia;
4. mantém a publicação sob `deployments/` e recusa links simbólicos nos caminhos relevantes;
5. compara os arquivos existentes antes de sobrescrevê-los;
6. executa outra validação sobre o resultado publicado.

> Trata-se de uma **simulação local**, e não de um sistema de deployment pronto para produção. A cópia é feita arquivo por arquivo; não há publicação atômica, rollback, controle de concorrência nem gerenciamento de segredos.

---

# 25. Conferindo a sintaxe da automação

Verifique os dois scripts:

```bash
bash -n "$LAB_DIR/scripts/preflight.sh"
bash -n "$LAB_DIR/scripts/deploy.sh"
```

Atribua permissão de execução:

```bash
chmod u+x "$LAB_DIR/scripts/deploy.sh"
```

Confirme:

```bash
ls -l "$LAB_DIR/scripts/preflight.sh" "$LAB_DIR/scripts/deploy.sh"
```

**Questão**

Por que o script chama `preflight.sh` antes e depois de copiar os arquivos?

---

# 26. Testando o modo `--dry-run`

Execute uma implantação simulada em `staging`:

```bash
APP_ENV=staging "$LAB_DIR/scripts/deploy.sh" "$LAB_DIR/app" release-01 --dry-run
```

A saída deverá incluir algo semelhante a:

```text
[OK] Preflight aprovado: ambiente=staging origem=...
[DRY-RUN] Publicaria .../app em .../deployments/staging/release-01
```

Comprove que o destino **não** foi criado:

```bash
if [[ -e "$LAB_DIR/deployments/staging/release-01" ]]; then
    printf '[ERRO] Dry-run modificou o destino.\n' >&2
else
    printf '[OK] Dry-run não criou a release.\n'
fi
```

> Como preparação e validação são operações de leitura, o *dry-run* pode executá-las normalmente. O que não pode acontecer é a alteração dos arquivos de destino.

---

# 27. Publicando uma release local

Agora execute sem `--dry-run`:

```bash
APP_ENV=staging "$LAB_DIR/scripts/deploy.sh" "$LAB_DIR/app" release-01
```

Saída esperada, em linhas semelhantes a:

```text
[OK] Preflight aprovado: ambiente=staging origem=...
[COPY] Arquivo publicado: app.txt
[COPY] Arquivo publicado: health.txt
[OK] Preflight aprovado: ambiente=staging origem=...
[OK] Release publicada: ambiente=staging release=release-01
```

Confira os arquivos:

```bash
find "$LAB_DIR/deployments/staging/release-01" -maxdepth 1 -type f | sort
```

Leia o conteúdo publicado:

```bash
cat "$LAB_DIR/deployments/staging/release-01/app.txt"
cat "$LAB_DIR/deployments/staging/release-01/health.txt"
```

Confirme que a origem e o destino possuem o mesmo conteúdo:

```bash
cmp "$LAB_DIR/app/app.txt" "$LAB_DIR/deployments/staging/release-01/app.txt"
cmp "$LAB_DIR/app/health.txt" "$LAB_DIR/deployments/staging/release-01/health.txt"
```

Quando os dois arquivos comparados são idênticos, `cmp` normalmente não imprime nada e retorna `0`.

---

# 28. Idempotência: executando novamente sem alterações

Execute a mesma publicação mais uma vez:

```bash
APP_ENV=staging "$LAB_DIR/scripts/deploy.sh" "$LAB_DIR/app" release-01
```

Agora, a saída deverá conter:

```text
[SKIP] Sem alterações: app.txt
[SKIP] Sem alterações: health.txt
```

O script reconhece que os arquivos de destino já possuem o conteúdo desejado e evita copiá-los novamente.

**Conceito**

Nesse cenário, executar repetidamente a mesma publicação mantém os arquivos no mesmo estado final. Isso demonstra uma forma simples de **idempotência**.

> A propriedade foi implementada para os arquivos controlados pelo exemplo. Ela não garante, por si só, a idempotência de operações arbitrárias de deployment, bancos de dados ou sistemas distribuídos.

---

# 29. Atualizando uma release de maneira controlada

Modifique o arquivo principal na origem:

```bash
printf 'Aplicação de demonstração - versão 2\n' > "$LAB_DIR/app/app.txt"
```

Execute novamente:

```bash
APP_ENV=staging "$LAB_DIR/scripts/deploy.sh" "$LAB_DIR/app" release-01
```

Observe as mensagens:

```text
[COPY] Arquivo publicado: app.txt
[SKIP] Sem alterações: health.txt
```

Confira a versão publicada:

```bash
cat "$LAB_DIR/deployments/staging/release-01/app.txt"
```

Resultado esperado:

```text
Aplicação de demonstração - versão 2
```

Perceba que o script não precisou copiar novamente `health.txt`, porque seu conteúdo permaneceu inalterado.

---

# 30. Verificando se entradas perigosas são rejeitadas

Teste um identificador de release inválido:

```bash
if APP_ENV=staging "$LAB_DIR/scripts/deploy.sh" "$LAB_DIR/app" "../fora-do-laboratorio"; then
    printf 'Resultado inesperado: sucesso.\n'
else
    printf '[OK] Identificador inválido foi bloqueado.\n'
fi
```

O script deve rejeitar o identificador antes de criar qualquer diretório.

Teste também um ambiente não permitido:

```bash
if APP_ENV=producao "$LAB_DIR/scripts/deploy.sh" "$LAB_DIR/app" release-02; then
    printf 'Resultado inesperado: sucesso.\n'
else
    printf '[OK] Ambiente inválido foi bloqueado.\n'
fi
```

**Reflexão**

Um script de implantação não deve aceitar indiscriminadamente caminhos ou entradas externas. Quando o programa realiza operações em arquivos, **validar os parâmetros reduz o risco de alterar recursos inesperados**.

---

# 31. Construindo uma pequena pipeline em Bash

Agora vamos encadear a checagem de sintaxe, a validação da origem, a publicação e a verificação do resultado.

Crie:

```bash
cat > "$LAB_DIR/scripts/pipeline.sh" <<'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

script_dir=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)
lab_dir=$(cd -- "$script_dir/.." && pwd -P)

release=${1:-release-pipeline}
ambiente=${APP_ENV:-dev}
origem="$lab_dir/app"

printf '[PIPELINE] 1/4 - Verificando sintaxe\n'
bash -n "$script_dir/preflight.sh" "$script_dir/deploy.sh"

printf '[PIPELINE] 2/4 - Validando origem\n'
"$script_dir/preflight.sh" "$origem"

printf '[PIPELINE] 3/4 - Publicando release\n'
"$script_dir/deploy.sh" "$origem" "$release"

printf '[PIPELINE] 4/4 - Verificando resultado\n'
"$script_dir/preflight.sh" "$lab_dir/deployments/$ambiente/$release"

printf '[PIPELINE] Concluída com sucesso.\n'
EOF
```

Torne-a executável:

```bash
chmod u+x "$LAB_DIR/scripts/pipeline.sh"
```

Execute:

```bash
APP_ENV=dev "$LAB_DIR/scripts/pipeline.sh" release-pipeline
```

Observe que a pipeline utiliza os códigos de saída dos scripts anteriores. Com `set -e`, uma falha não tratada em uma dessas etapas interrompe a execução.

> Isso é uma **pipeline didática executada localmente**, não uma integração com GitHub Actions, GitLab CI ou outra plataforma. Nos módulos de CI/CD, ferramentas especializadas executarão fluxos equivalentes em ambientes controlados.

---

# 32. Validando uma falha dentro da pipeline

Vamos provocar uma falha funcional **controlada** na origem:

```bash
printf 'unhealthy\n' > "$LAB_DIR/app/health.txt"
```

Execute a pipeline tratando a falha esperada:

```bash
if APP_ENV=dev "$LAB_DIR/scripts/pipeline.sh" release-com-falha; then
    printf 'Resultado inesperado: a pipeline aprovou a origem inválida.\n'
else
    printf '[OK] Pipeline interrompida por falha no preflight.\n'
fi
```

Ela deverá parar na validação da origem, **antes da publicação**.

Confirme que a release não foi criada:

```bash
if [[ -e "$LAB_DIR/deployments/dev/release-com-falha" ]]; then
    printf '[ERRO] A release inválida foi criada.\n' >&2
else
    printf '[OK] Release inválida não foi publicada.\n'
fi
```

Restaure a condição de saúde:

```bash
printf 'healthy\n' > "$LAB_DIR/app/health.txt"
```

Repita a pipeline com uma release válida:

```bash
APP_ENV=dev "$LAB_DIR/scripts/pipeline.sh" release-recuperada
```

Essa sequência reproduz, em pequena escala, um princípio operacional importante: **uma etapa de validação deve impedir a continuação de uma entrega inválida**.

---

# 33. Registrando evidências de execução

Execute a pipeline e grave suas mensagens em um arquivo:

```bash
APP_ENV=dev "$LAB_DIR/scripts/pipeline.sh" release-auditada \
    > "$LAB_DIR/logs/pipeline.log" 2>&1
```

Confira os registros:

```bash
cat "$LAB_DIR/logs/pipeline.log"
```

Filtre apenas as etapas da pipeline:

```bash
grep '^\[PIPELINE\]' "$LAB_DIR/logs/pipeline.log"
```

O registro deverá conter as quatro etapas e a mensagem de conclusão.

> Ao redirecionar `2>&1` após `> arquivo`, unificamos `stderr` e `stdout` no mesmo arquivo. Em ambientes reais, o conteúdo dos logs deve ser planejado para evitar exposição de dados sensíveis.

---

# 34. Diagnóstico e análise dos scripts

Verifique a sintaxe dos scripts criados:

```bash
for script in "$LAB_DIR"/scripts/*.sh; do
    if bash -n "$script"; then
        printf '[OK] Sintaxe: %s\n' "$script"
    else
        printf '[ERRO] Sintaxe inválida: %s\n' "$script" >&2
    fi
done
```

Se nenhum erro for apresentado, a **sintaxe** está correta. Isso ainda não substitui os testes de comportamento anteriores.

Faça um teste de rastreamento de execução em um script que não utiliza credenciais:

```bash
bash -x "$LAB_DIR/scripts/preflight.sh" "$LAB_DIR/app" \
    > "$LAB_DIR/logs/preflight-stdout.log" \
    2> "$LAB_DIR/logs/preflight-trace.log"
```

Confira algumas linhas:

```bash
sed -n '1,20p' "$LAB_DIR/logs/preflight-trace.log"
```

O `bash -x` mostra comandos durante sua execução e pode ajudar a localizar o ponto de falha.

> **Atenção:** esse modo também pode expor valores de variáveis e argumentos. Não o ative indiscriminadamente em scripts que lidam com segredos.

## Análise estática opcional

Se a ferramenta `shellcheck` estiver instalada, execute:

```bash
if command -v shellcheck >/dev/null 2>&1; then
    shellcheck "$LAB_DIR/scripts/preflight.sh" \
               "$LAB_DIR/scripts/deploy.sh" \
               "$LAB_DIR/scripts/pipeline.sh"
else
    printf 'ShellCheck não instalado: etapa opcional ignorada.\n'
fi
```

O ShellCheck analisa o código sem executá-lo e pode apontar construções potencialmente problemáticas. **Um aviso não significa necessariamente que o script falhou; avalie o contexto antes de alterar o código.**

---

# 35. Desafio: evoluindo a automação

Você recebeu a seguinte solicitação de uma equipe de Engenharia de Software:

> Precisamos de um procedimento que verifique se os arquivos necessários estão corretos, publique uma versão em um ambiente de teste e avise claramente quando alguma etapa falhar. Reexecutar a mesma operação não deve provocar cópias desnecessárias.

Usando os scripts construídos, responda:

1. Que argumentos e variáveis de ambiente controlam a execução?
2. Como o script diferencia `dev` de `staging`?
3. Qual validação impede que `unhealthy` seja publicado?
4. O que `--dry-run` permite conferir antes de modificar os arquivos?
5. Em quais situações o script retorna um código diferente de zero?
6. Como o `for` contribui para verificar e publicar os arquivos?
7. Por que `"$@"`, aspas em caminhos e validação de parâmetros são importantes?
8. O que acontece quando a mesma release é executada novamente sem alterações?
9. Qual evidência demonstra que os arquivos publicados são equivalentes aos arquivos de origem?
10. Por que a pipeline interrompe a publicação quando o *preflight* falha?
11. O que faltaria para transformar essa simulação em um deployment de produção seguro?

## Desafio adicional (opcional)

Adapte o *preflight* para exigir também um arquivo `config.txt` **não vazio**. Depois:

1. verifique que a aplicação sem esse arquivo é rejeitada;
2. crie o arquivo esperado no diretório `app/`;
3. ajuste `deploy.sh` para copiá-lo e compará-lo;
4. confirme que a publicação e a validação final continuam funcionando;
5. execute novamente a mesma release e confirme os `[SKIP]` nos três arquivos.

> Esse desafio opcional altera os scripts. As saídas descritas anteriormente pressupõem a versão original do laboratório, com dois arquivos obrigatórios.

---

# 36. Limpando o ambiente

**Antes de remover qualquer arquivo**, confira qual diretório está configurado:

```bash
printf '%s\n' "$LAB_DIR"
```

Ele deverá corresponder exatamente a:

```text
<seu-diretório-pessoal>/lab-devops-bash-04
```

Liste o conteúdo que será removido:

```bash
find "$LAB_DIR" -type f | sort
```

Se tiver terminado o laboratório e quiser remover **somente** os arquivos criados aqui, execute:

```bash
if [[ "$LAB_DIR" == "$HOME/lab-devops-bash-04" && -d "$LAB_DIR" && ! -L "$LAB_DIR" ]]; then
    rm -r -- "$LAB_DIR"
    printf '[OK] Diretório do laboratório removido.\n'
else
    printf '[ERRO] Caminho não corresponde ao laboratório; nada removido.\n' >&2
fi
```

Essa verificação protege contra a remoção acidental causada por uma variável vazia ou com caminho inesperado. **Não execute esse comando se desejar manter os arquivos para estudar ou entregar o exercício.**

Se quiser, remova do terminal apenas as variáveis criadas na prática:

```bash
unset APP_NAME APP_VERSION APP_ENV DATA_EXECUCAO CONTADOR GUIA LAB_MARCA
unset LAB_DIR
```

---

# 37. Checklist final

Ao concluir o laboratório, você deve conseguir explicar a diferença entre:

```text
Bash × Linux

comando interativo × Shell Script

execução direta × execução com bash × source

variável da shell × variável de ambiente

variável de ambiente × argumento

aspas simples × aspas duplas

expansão de variável × substituição de comando

stdout × stderr × código de saída

sucesso (0) × falha (≠ 0)

if × case

for × while

script principal × função

validação de sintaxe × validação de comportamento

dry-run × execução real

execução repetida × operação idempotente

publicação concluída × resultado validado
```

Também deve ser capaz de identificar, nos scripts desenvolvidos, onde acontecem:

```text
Entrada
   ↓
Validação
   ↓
Decisão
   ↓
Execução
   ↓
Verificação
   ↓
Resultado / Código de saída
```

---

# 38. Resultado esperado

O objetivo não é memorizar todas as construções da linguagem Bash. O objetivo é saber transformar um procedimento operacional em um programa que:

```text
Recebe informações de maneira clara
                ↓
Valida condições antes de agir
                ↓
Executa somente as operações necessárias
                ↓
Comunica erros de forma interpretável
                ↓
Verifica se o estado esperado foi atingido
                ↓
Pode ser repetido de maneira previsível
```

Esses princípios continuarão importantes quando os próximos módulos apresentarem **controle de versão, CI/CD, containers, infraestrutura como código e automação de ambientes**.

> **Limites do exercício:** a validação utiliza arquivos locais; a publicação não é atômica; não há rollback, isolamento entre executores, controle de concorrência, gestão de credenciais nem health check de rede. Esses mecanismos serão relevantes em automações de produção, mas não fazem parte do escopo deste laboratório introdutório.

---

# Referências

**NEMETH, E.; SNYDER, G.; HEIN, T. R.; WHALEY, B.; MACKIN, D. UNIX and Linux System Administration Handbook. 5. ed. Pearson, 2018. Cap. 7 — Scripting and the Shell (especialmente p. 183–209).**

**KIM, G.; HUMBLE, J.; DEBOIS, P.; WILLIS, J.; FORSGREN, N. The DevOps Handbook: How to Create World-Class Agility, Reliability, & Security in Technology Organizations. 2. ed. IT Revolution Press, 2021. Parte III, Caps. 9 — Create the Foundations of Our Deployment Pipeline e 12 — Automate and Enable Low-Risk Releases.**

**FORSGREN, N.; HUMBLE, J.; KIM, G. Accelerate: The Science of Lean Software and DevOps. IT Revolution Press, 2018. Cap. 4 — Technical Practices.**
