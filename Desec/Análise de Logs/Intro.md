## Análise de logs I

Primeiro baixei o arquivo de log para o diretório atual.

```
wget www.businesscorp.com.br/access.log
```

Assim que tive acesso aos logs, conferi a quantidade de linhas

```
wc -l access.log
```

Então, conferi o padrão de cada linha

```
head -n1 access.log
```

Para pegar os ips foi preciso usar

```
cat access.log | cut -d " " -f 1 | sort -u
```

cut com delimitador " ", e apenas na primeira coluna, com um sort -u para pegar os únicos, para saber a quantidade de ips diferentes

```
Resposta Análise de log I: setentaenvoe
```

## Análise de logs II

```
cat access.log | cut -d " " -f 1 | sort | uniq -c
```
sort sem -u para pegar os repetidos, e unic -c para pegar a quantidade de cada um. Assim sabemos qual foi o ID do atacante.

```
Resposta Análise de log II: 177.138.28.7
```

## Análise de logs III

Para achar o início e o fim do ataque

```
cat access.log | grep "177.138.28.7" | head -n1
```
```
cat access.log | grep "177.138.28.7" | tail -n1
```

Formato de resposta
```
13/Feb/2015:00:21:12,13/Feb/2015:08:58:35
```