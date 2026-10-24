# Load balancer com nginx

Kit de treinamento para subir, na sua própria máquina, dois sites HTML e um load balancer na frente deles. O usuário acessa uma URL única e o nginx decide qual dos dois sites responde.

```
+-----------+              +-----------------+              +---------------------+
|           |    :4000     |                 |    :5001     | Servidor 1  (site1) |
| Navegador | -----------> |  Load balancer  | -----------> | página azul         |
|           |              |     (nginx)     |              +---------------------+
+-----------+              |                 |    :5002     +---------------------+
                           |                 | -----------> | Servidor 2  (site2) |
                           +-----------------+              | página amarela      |
                                                            +---------------------+
```

Tudo roda em um único processo do nginx: ele é o load balancer na porta 4000 e, ao mesmo tempo, o servidor web de cada site nas portas 5001 e 5002. Em um ambiente real cada caixa do desenho seria uma máquina diferente. Aqui está tudo junto para você focar no balanceamento.

## O que você vai fazer

1. Instalar o nginx.
2. Apontar o `nginx.conf` para as pastas `site1` e `site2`.
3. Subir tudo com um comando.
4. Ver, no navegador e no terminal, a alternância entre os servidores.
5. Trocar o algoritmo de balanceamento e observar o efeito.

## Por que existe um load balancer

Quando um site cresce, um servidor só não dá conta. A solução é ter várias cópias do site em servidores diferentes e colocar um **load balancer** na frente. Ele recebe todas as requisições em um único endereço e as distribui entre as cópias. Para o usuário nada muda: ele continua acessando uma URL só.

O algoritmo padrão de distribuição é o **Round-Robin**: a primeira requisição vai para o servidor 1, a segunda para o servidor 2, a terceira volta para o 1, e assim por diante. Nos exercícios você vai testar outros algoritmos.

## O que tem neste repositório

| Arquivo | Para que serve |
|---|---|
| `nginx.conf` | Configuração do load balancer e dos dois servidores. É o arquivo que você vai editar. |
| `site1/index.html` | Página azul que identifica o servidor 1. |
| `site2/index.html` | Página amarela que identifica o servidor 2. |
| `LICENSE` | Licença MIT. Use e modifique à vontade. |

## Pré-requisitos

- Git.
- nginx instalado. Siga a seção do seu sistema.

### Windows

1. Baixe o nginx para Windows em https://nginx.org/en/download.html (arquivo `.zip` da versão *Mainline*).
2. Descompacte e renomeie a pasta para `C:\nginx`. Dentro dela deve existir o `nginx.exe`.
3. Abra o **Prompt de Comando** (cmd) e entre na pasta: `cd C:\nginx`.

Todos os comandos do nginx deste roteiro devem ser executados no cmd, de dentro de `C:\nginx`.

### macOS

```bash
brew install nginx
```

### Linux (Ubuntu/Debian)

```bash
sudo apt update && sudo apt install nginx
```

O pacote já sobe um nginx padrão na porta 80. Pare-o antes de usar o do kit:

```bash
sudo systemctl stop nginx
```

No Linux, todos os comandos do nginx deste roteiro precisam de `sudo` na frente.

### Conferindo a instalação

```bash
nginx -v
```

Deve mostrar algo como `nginx version: nginx/1.29.2`.

## Passo a passo

### 1. Clone o repositório

```bash
git clone https://github.com/leopellizzoni/load-balancer.git
cd load-balancer
```

Anote o caminho completo da pasta. Você vai precisar dele nos próximos passos.

- Windows: `cd` sem argumentos mostra o caminho. Exemplo: `C:\Users\maria\load-balancer`.
- macOS e Linux: `pwd`. Exemplo: `/Users/maria/load-balancer`.

### 2. Aponte o nginx.conf para as suas pastas

Abra o `nginx.conf` em um editor de texto. Troque `CAMINHO_DO_REPOSITORIO` (aparece duas vezes, nas linhas `root`) pelo caminho que você anotou. Use barras normais `/` mesmo no Windows.

Antes:

```nginx
root CAMINHO_DO_REPOSITORIO/site1;
```

Depois, por exemplo no Windows:

```nginx
root C:/Users/maria/load-balancer/site1;
```

Se o caminho tiver espaços, coloque entre aspas: `root "C:/Meus Projetos/load-balancer/site1";`

### 3. Teste a configuração

O nginx consegue verificar o arquivo sem subir nada. Passe o caminho completo do `nginx.conf`:

macOS:

```bash
nginx -t -c /Users/maria/load-balancer/nginx.conf
```

Linux:

```bash
sudo nginx -t -c /home/maria/load-balancer/nginx.conf
```

Windows (cmd, dentro de `C:\nginx`):

```
nginx -t -c C:/Users/maria/load-balancer/nginx.conf
```

Resultado esperado: duas linhas terminando em `syntax is ok` e `test is successful`. Se aparecer `[emerg]`, leia a mensagem: ela diz o arquivo e a linha do problema.

Atenção: o teste não confere se as pastas das linhas `root` existem. Um caminho errado só aparece no passo 5, como erro 404.

### 4. Suba o nginx

É o mesmo comando do passo 3, sem o `-t`:

macOS:

```bash
nginx -c /Users/maria/load-balancer/nginx.conf
```

Linux:

```bash
sudo nginx -c /home/maria/load-balancer/nginx.conf
```

Windows (cmd, dentro de `C:\nginx`):

```
start nginx -c C:/Users/maria/load-balancer/nginx.conf
```

Quando dá certo o comando não mostra nada e o nginx fica rodando em segundo plano.

### 5. Confira cada servidor sozinho

Antes de testar o balanceador, confirme que os dois sites estão no ar:

- http://localhost:5001 deve mostrar a página azul **Servidor 1**.
- http://localhost:5002 deve mostrar a página amarela **Servidor 2**.

Se aparecer `404 Not Found`, o caminho do passo 2 está errado. Corrija e recarregue com `nginx -s reload`.

### 6. Acesse pela URL única

Agora abra http://localhost:4000 e aperte **F5** várias vezes.

A página deve alternar: azul, amarela, azul, amarela. Cada F5 é uma requisição nova, e o load balancer manda uma para cada servidor, em rodízio. Esse é o Round-Robin funcionando.

### 7. Confira pelo terminal

Cada servidor devolve um cabeçalho HTTP chamado `X-Servidor` dizendo quem respondeu. Dispare 10 requisições seguidas e mostre só esse cabeçalho:

macOS e Linux:

```bash
for i in {1..10}; do curl -sI localhost:4000 | grep X-Servidor; done
```

Windows (cmd):

```
for /L %i in (1,1,10) do @curl -sI localhost:4000 | findstr X-Servidor
```

Resultado esperado: `servidor1` e `servidor2` se alternando linha a linha.

Se estiver no PowerShell em vez do cmd, use `curl.exe` no lugar de `curl`: no PowerShell, `curl` é um apelido de outro comando.

### 8. Para parar

macOS: `nginx -s stop`

Linux: `sudo nginx -s stop`

Windows (cmd, dentro de `C:\nginx`): `nginx -s stop`

Sempre que alterar o `nginx.conf` com o nginx no ar, aplique a mudança com `nginx -s reload` (com `sudo` no Linux). Não precisa parar e subir de novo.

## Exercícios

Todos seguem o mesmo ciclo: edite o `nginx.conf`, rode `nginx -s reload`, repita o passo 7 e anote o que mudou. As linhas necessárias já estão comentadas dentro do bloco `upstream` do `nginx.conf`.

### 1. Pesos (Weighted Round-Robin)

No bloco `upstream`, troque:

```nginx
server 127.0.0.1:5001;
```

por:

```nginx
server 127.0.0.1:5001 weight=2;
```

Recarregue e dispare 9 requisições. Quantas foram para cada servidor? Em que situação faz sentido dar peso maior a um servidor?

### 2. Sessão fixa (ip_hash)

Desfaça o exercício anterior. Adicione `ip_hash;` como primeira linha dentro do bloco `upstream`. Recarregue e aperte F5 várias vezes em http://localhost:4000.

O que aconteceu com a alternância? O `ip_hash` leva o mesmo cliente sempre para o mesmo servidor. Pense em uma situação em que isso é necessário. Dica: um carrinho de compras guardado na memória do servidor.

### 3. Menos conexões (least_conn)

Troque o `ip_hash;` por `least_conn;`. Recarregue e repita o passo 7.

Você provavelmente não vai notar diferença em relação ao Round-Robin. Por quê? Cada página aqui é entregue em milissegundos, então nunca há conexões acumuladas em um servidor. Em que tipo de sistema o `least_conn` faria diferença?

### 4. Terceiro servidor

Remova o `least_conn;` para voltar ao Round-Robin. Depois:

1. Copie a pasta `site2` para `site3` e mude o texto e a cor do `index.html` para identificar o servidor 3.
2. No `nginx.conf`, copie o bloco `server` do site2, trocando a porta para 5003, a pasta para `site3` e o cabeçalho para `servidor3`.
3. Adicione `server 127.0.0.1:5003;` no bloco `upstream`.
4. Recarregue e repita o passo 7.

Esse é o ponto principal de um load balancer: você aumentou a capacidade sem mudar a URL que o usuário acessa.

### 5. Simule a queda de um servidor

Com os três servidores no ar, comente o bloco `server` inteiro do site2 (coloque `#` no início de cada linha) e recarregue. A porta 5002 deixa de existir, como se a máquina tivesse desligado.

Aperte F5 em http://localhost:4000 várias vezes. Apareceu algum erro? O que o load balancer fez com as requisições que iriam para o servidor 2?

Depois descomente o bloco e recarregue. Observe se o servidor 2 voltou ao rodízio.

Variação: em vez de comentar o bloco, marque o servidor no `upstream` com `server 127.0.0.1:5002 down;`. É assim que se tira um servidor do ar de propósito, para manutenção, sem derrubar nada.

## Problemas comuns

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `nginx: [emerg] open() "...nginx.conf" failed` | Caminho do `-c` errado | Confira o caminho completo do arquivo. Precisa ser absoluto. |
| `404 Not Found` em :5001 ou :5002 | `CAMINHO_DO_REPOSITORIO` não foi trocado ou está errado | Refaça o passo 2 e rode `nginx -s reload`. |
| `403 Forbidden` no Linux | O nginx não tem permissão para entrar na sua pasta home | Rode `chmod o+x ~` ou clone o repositório em `/opt` ou `/srv`. |
| `bind() to 0.0.0.0:4000 failed (Address already in use)` | O nginx do kit já está rodando, ou outro programa usa a porta | Se o nginx já estiver no ar, use `nginx -s reload`. Senão, troque a porta no `nginx.conf`. |
| Alterei o `nginx.conf` e nada mudou | Faltou aplicar a mudança | Rode `nginx -s reload`. |
| A página não alterna no navegador | Cache do navegador ou extensão interferindo | Teste pelo terminal (passo 7). Se alterna lá, abra uma aba anônima. |
| `nginx -s stop` reclama que não achou o pid | O nginx não está rodando | Nada a fazer. |
| Windows: `curl` não é reconhecido ou dá erro estranho | Você está no PowerShell, onde `curl` é outro comando | Use o cmd, ou troque `curl` por `curl.exe`. |
| Windows: `nginx -s stop` não funciona | Comando executado fora de `C:\nginx` | Entre na pasta com `cd C:\nginx` e repita. |

## Referência rápida de comandos

| O que faz | macOS | Linux | Windows (cmd, em `C:\nginx`) |
|---|---|---|---|
| Testar a configuração | `nginx -t -c /caminho/nginx.conf` | `sudo nginx -t -c /caminho/nginx.conf` | `nginx -t -c C:/caminho/nginx.conf` |
| Subir | `nginx -c /caminho/nginx.conf` | `sudo nginx -c /caminho/nginx.conf` | `start nginx -c C:/caminho/nginx.conf` |
| Aplicar mudança na configuração | `nginx -s reload` | `sudo nginx -s reload` | `nginx -s reload` |
| Parar | `nginx -s stop` | `sudo nginx -s stop` | `nginx -s stop` |
| Ver a versão | `nginx -v` | `nginx -v` | `nginx -v` |

## Entendendo o nginx.conf

O arquivo tem dois blocos de primeiro nível. `events {}` controla como o nginx trata conexões e pode ficar vazio aqui. Tudo que interessa está dentro de `http {}`.

- **`http`**: contexto principal para tudo que é HTTP. Os servidores virtuais (`server`) e o grupo de balanceamento (`upstream`) ficam aqui dentro.
- **`upstream meus_sites`**: o grupo de servidores entre os quais as requisições serão distribuídas. Cada linha `server` é um destino. Opções como `weight`, `down`, `least_conn` e `ip_hash` mudam a forma de distribuir.
- **`server`**: um servidor virtual. Há três no arquivo: o load balancer (porta 4000) e os dois sites (portas 5001 e 5002).
- **`listen`**: a porta em que aquele `server` recebe conexões.
- **`server_name`**: o nome de host que aquele `server` atende. Aqui é sempre `localhost`.
- **`location /`**: captura as requisições que chegam no `server`. O `/` casa com qualquer URL.
- **`proxy_pass`**: repassa a requisição recebida para o grupo `meus_sites`. É a diretiva que transforma o `server` da porta 4000 em load balancer.
- **`root`**: a pasta onde estão os arquivos que o `server` entrega. É a linha que você edita no passo 2.
- **`index`**: o arquivo entregue quando a URL termina em `/`. Aqui é o `index.html`.
- **`add_header`**: acrescenta um cabeçalho HTTP na resposta. Usado para o `X-Servidor` (identificar quem respondeu) e para o `Cache-Control` (impedir que o navegador guarde a página e esconda a alternância).

## Licença

MIT. Veja o arquivo [LICENSE](LICENSE).
