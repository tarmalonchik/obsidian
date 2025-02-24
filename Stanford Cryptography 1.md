### Terminology 
+ eavesdropping - подслушивание
+ tampering - вмешательство

## Симметричное шифрование
+ E - encryption fuction
+ D - decryption function
+ m - plain text
+ k - secret key
+ c - cypher text

Алиса делает это:
`E(k,m) = c`
Бот делает это: 
`D(k,c) = m`

E, D is public information
> Never use a proprietary algorighm

#### Use cases 
+ Single use key
+ Multi use key 