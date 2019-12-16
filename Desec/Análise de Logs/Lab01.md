## Lab01
IP do atacante

Ao analisar o log http://www.businesscorp.com.br/lab.log com 
```
cat lab.log | cut -d " " -f 1 | sort | uniq -c
```

O IP de maior incidência é

```
205.251.150.186
```