## Investigando o Ataque - Lab2
Examine o arquivo de log e responda em qual diretório o atacante encontrou a informação sensível?

Exemplo: uploads

http://www.businesscorp.com.br/lab.log

---
Além descobri o IP no lab01, segui investigando

```
cat lab.log | grep "205.251.150.186" | cut -d "]" -f2 | grep "\" 200" | sort -u
```

Analisando o resultado, que vinheram apenas as requisições com 200, testei cada uma das rotas e testei as repostas. A correta era

```
configuracoes
```