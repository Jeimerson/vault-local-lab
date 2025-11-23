# Vault para Desenvolvimento Local

Esta é uma configuração do Docker Compose para trabalho de desenvolvimento usando Vault e Consul.

* FORKED de:

  * [https://github.com/tolitius/cault](https://github.com/tolitius/cault)
  * [https://github.com/spkane/vault-local-dev](https://github.com/spkane/vault-local-dev)

## Iniciando Consul e Vault

```
export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_CAPATH="${PWD}/certs/ca.crt"
docker compose up -d
```

* [Vault UI](https://127.0.0.1:8200/ui)
* [Consul UI](http://127.0.0.1:8500/ui)

## Preparando o Vault

* **NOTA**: É uma boa prática usar a mesma versão do CLI do Vault que estamos usando para o servidor Vault!

### Bootstrap

Você pode inicializar o Vault via [Vault UI](https://127.0.0.1:8200/ui) ou linha de comando.

### Linha de Comando

* Execute `vault operator init` para criar as chaves de unseal e o token root inicial.

  * Se você ver `* Vault is already initialized`, significa que já fez isso antes.
  * Anote estas informações! Se perder, precisará reiniciar o processo.
* Execute `vault operator unseal` três vezes seguidas, fornecendo ao Vault uma das 5 chaves de unseal diferentes.

  * A saída conterá uma linha que começa com `Unseal Progress`. Você quer que essa linha desapareça completamente e que a linha `Sealed` mostre `false`.
* Após o Vault ser deslacrado (unsealed), execute `vault login` com o Token Root Inicial obtido ao executar `vault operator init` anteriormente.

### Backup & Restore

* **NOTA**: Este container não persiste NENHUM dado, pois o nó do Consul está em modo dev.

* `docker compose pause` e `docker compose unpause` são as únicas maneiras de parar os containers sem perder os dados, sem necessidade de backup. Isso não sobrevive a um reboot, então você precisará restaurar e deslacrar após os containers pararem.

#### Backup

* Para salvar os dados para uso posterior, instale o cliente Consul localmente e execute:

  * `consul snapshot save backups/vault-consul-backup-$(date +%Y%m%d%H%M).snap`

#### Restore

* Para restaurar, use algo como:

  * `consul snapshot restore ./backups/vault-consul-backup-202110021214.snap`

## README do repositório forked:

Consul e Vault são iniciados juntos em dois containers Docker separados, mas ligados.

Vault é configurado para usar um [backend de secrets do Consul](https://www.vaultproject.io/docs/secrets/consul/).

---

* [Vault para Desenvolvimento Local](#vault-para-desenvolvimento-local)

  * [Iniciando Consul e Vault](#iniciando-consul-e-vault)
  * [Preparando o Vault](#preparando-o-vault)

    * [Bootstrap](#bootstrap)
    * [Linha de Comando](#linha-de-comando)
    * [Backup & Restore](#backup--restore)

      * [Backup](#backup)
      * [Restore](#restore)
  * [README do repositório forked:](#readme-do-repositorio-forked)
  * [Iniciando Consul e Vault](#iniciando-consul-e-vault-1)
  * [Preparando o Vault](#preparando-o-vault-1)

    * [Inicializando o Vault](#inicializando-o-vault)
    * [Deslacrando o Vault](#deslacrando-o-vault)
    * [Autenticação no Vault](#autenticacao-no-vault)
  * [Verificando se realmente funciona](#verificando-se-realmente-funciona)

    * [Acompanhar logs do Consul](#acompanhar-logs-do-consul)
    * [Escrevendo / Lendo Secrets](#escrevendo--lendo-secrets)
    * [Response Wrapping](#response-wrapping)

      * [System backend](#system-backend)
      * [Cubbyhole backend](#cubbyhole-backend)
  * [Solução de Problemas](#solucao-de-problemas)

    * [Caches de Imagens Ruins](#caches-de-imagens-ruins)
  * [License](#license)

## Iniciando Consul e Vault

```bash
docker-compose up -d
```

## Preparando o Vault

Login na imagem do Vault:

```bash
docker exec -it cault_vault_1 sh
```

Verifique o status do Vault:

```bash
$ vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            n/a
HA Enabled         true
```

Como o Vault ainda não está inicializado (`Initialized  false`), ele está lacrado (`Sealed  true`), por isso o Consul mostrará status crítico lacrado:

<p align="center"><img src="doc/img/sealed-vault.png"></p>

### Inicializando o Vault

```bash
$ vault operator init
Unseal Key 1: dW2PXpPdjWZvXCUvE/GWxJ+CdeEp6SziEKh6xNYRpB8k
Unseal Key 2: 5K52IOOU+rZf+6Aj7PBOTclnL80Ftb1Wta1GbrJDWX8f
Unseal Key 3: ykK/Q5Il7OOp/qKTdT75U1q6EDzMo2LkM0KRWv7I11Lb
Unseal Key 4: /1EVEn1UDG4LbqI2h5MQPWRI1wpCbirELJyVBo+D2QR1
Unseal Key 5: H47Vch2d0AxuA43kxOlW+MzC/YtjoGU8wCoZLDmRg29r

Initial Root Token: s.1ee2zxWvX43sAwjlcDaSGGSC

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

Repare que o Vault diz:

> você deve fornecer pelo menos 3 dessas chaves para deslacrar novamente

Portanto, ele precisa ser deslacrado 3 vezes com 3 chaves diferentes (das 5 acima).

### Deslacrando o Vault

```bash
$ vault operator unseal
Unseal Key (will be hidden):
Key                Value
---                -----
...
Sealed             true
Unseal Progress    1/3

$ vault operator unseal
Unseal Key (will be hidden):
Key                Value
---                -----
...
Sealed             true
Unseal Progress    2/3

$ vault operator unseal
Unseal Key (will be hidden):
Key                    Value
---                    -----
...
Initialized            true
Sealed                 false
...
Active Node Address    <none>
```

O Vault agora está deslacrado:

<p align="center"><img src="doc/img/unsealed-vault.png"></p>

### Autenticação no Vault

Podemos usar o `Initial Root Token` acima para autenticar no Vault:

```bash
$ vault login
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.1ee2zxWvX43sAwjlcDaSGGSC
token_accessor       shMBI822edbRUYTo8mW54mdB
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

---

Tudo pronto: agora você tem Consul e Vault rodando lado a lado.

## Verificando se realmente funciona

Do ambiente host (ou seja, fora da imagem Docker):

```bash
alias vault='docker exec -it cault_vault_1 vault "$@"'
```

Isso permitirá rodar comandos `vault` sem precisar logar na imagem.

> O motivo pelo qual os comandos funcionarão é que você acabou de se autenticar com um token root dentro da imagem no passo anterior.

### Acompanhar logs do Consul

Em um terminal, acompanhe os logs do Consul:

```bash
$ docker logs cault_consul_1 -f
```

### Escrevendo / Lendo Secrets

No outro terminal, rode comandos do Vault:

```bash
$ vault write -address=http://127.0.0.1:8200 cubbyhole/billion-dollars value=behind-super-secret-password
```

```
Success! Data written to: cubbyhole/billion-dollars
```

Confira o log do Consul, você verá algo como:

```bash
2016/12/28 06:52:09 [DEBUG] http: Request PUT /v1/kv/vault/logical/a77e1d7f-a404-3439-29dc-34a34dfbfcd2/billion-dollars (199.657µs) from=172.28.0.3:50260
```

Vamos ler de volta:

```bash
$ vault read cubbyhole/billion-dollars
```

```
Key             	Value
---             	-----
value           	behind-super-secret-password
```

E de fato está no Consul:

<p align="center"><img src="doc/img/vault-value-in-consul.png"></p>

E no Vault:

<p align="center"><img src="doc/img/secret-in-vault-ui.png"></p>

(isto é da própria UI do Vault que está habilitada nesta imagem)

### Response Wrapping

> *NOTA: para estes exemplos funcionarem, você precisará do [jq](https://stedolan.github.io/jq/) (para parse de respostas JSON do Vault).*

> *`brew install jq` ou `apt-get install jq` ou similar*

#### System backend

Rodando com um [System Secret Backend](https://www.vaultproject.io/api/system/index.html).

Exporte variáveis do Vault para que scripts locais funcionem:

```bash
$ export VAULT_ADDR=http://127.0.0.1:8200
$ export VAULT_TOKEN=s.1ee2zxWvX43sAwjlcDaSGGSC  ### token root que você guardou ao inicializar o Vault
```

Na raiz do projeto `cault` há um arquivo `creds.json` (você pode criar o seu próprio):

```bash
$ cat creds.json

{"username": "ceo",
 "password": "behind-super-secret-password"}
```

Podemos escrever em um "local de uso único" no Vault. Esse local será acessível por um "token de uso único" retornado pelo Vault no endpoint `/sys/wrapping/wrap`:

```bash
$ token=`./tools/vault/wrap-token.sh creds.json`

$ echo $token
s.sMFwpg8DBYh0NXbXqjLJTNKN
```

Você pode conferir o script [wrap-token.sh](tools/vault/wrap-token.sh), que usa o endpoint `/sys/wrapping/wrap` do Vault para persistir secretamente `creds.json` e retorna um token válido por 60 segundos.

Agora vamos usar este token para "desembrulhar" o secret:

```bash
$ ./tools/vault/unwrap-token.sh $token

{"password": "behind-super-secret-password",
 "username": "ceo" }
```

Você pode conferir o script [unwrap-token.sh](tools/vault/unwrap-token.sh), que usa o endpoint `/sys/wrapping/unwrap`.

Vamos tentar usar o mesmo token novamente:

```bash
$ ./tools/vault/unwrap-token.sh $token
["wrapping token is not valid or does not exist"]
```

Ou seja, o Vault leva o "uso único" muito a sério.

#### Cubbyhole backend

Rodando com um [Cubbyhole Secret Backend](https://www.vaultproject.io/docs/secrets/cubbyhole/index.html).

Exporte variáveis do Vault para que scripts locais funcionem:

```bash
$ export VAULT_ADDR=http://127.0.0.1:8200
$ export VAULT_TOKEN=s.1ee2zxWvX43sAwjlcDaSGGSC  ### token root que você guardou ao inicializar o Vault
```

Crie um cubbyhole para o secret `billion-dollars` e envolva em um token de uso único:

```bash
$ token=`./tools/vault/cubbyhole-wrap-token.sh /cubbyhole/billion-dollars`
```

Vamos olhar:

```bash
$ echo $token
s.T3GT2dGb8bUuJtSEenxnZick
```

Parece um token comum, mas é de *uso único*, apenas para este cubbyhole.

Vamos usar:

```bash
$ curl -s -H "X-Vault-Token: $token" -X GET $VAULT_ADDR/v1/cubbyhole/response
```

```json
{
  "request_id": "f0cf41a6-d971-69be-4eee-c7137376a755",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "response": "{\"request_id\":\"083429a1-2956-39f0-a402-628b6e346ac0\",\"lease_id\":\"\",\"renewable\":false,\"lease_duration\":0,\"data\":{\"value\":\"behind-super-secret-password\"},\"wrap_info\":null,\"warnings\":null,\"auth\":null}"
  },
  "wrap_info": null,
  "warnings": [
    "Reading from 'cubbyhole/response' is deprecated. Please use sys/wrapping/unwrap to unwrap responses, as it provides additional security checks and other benefits."
  ],
  "auth": null
}
```

*(observe: `cubbyhole/response` está obsoleto, use o backend `system` como no exemplo acima)*

Vamos tentar usar novamente:

```bash
$ curl -s -H "X-Vault-Token: $token" -X GET $VAULT_ADDR/v1/cubbyhole/response
```

```json
{"errors":["permission denied"]}
```

O Vault leva o "uso único" muito a sério.

## Solução de Problemas

### Caches de Imagens Ruins

Caso existam imagens antigas/paradas em cache, você pode obter exceções de conexão:

```clojure
failed to check for initialization: Get v1/kv/vault/core/keyring: dial tcp i/o timeout
```

```clojure
reconcile unable to talk with Consul backend: error=service registration failed: /v1/agent/service/register
```

Você pode limpar imagens paradas para resolver:

```bash
docker rm $(docker ps -a -q)
```

## License

Copyright © 2019 tolitius

Distribuído sob a Eclipse Public License, versão 1.0 ou (a seu critério) qualquer versão posterior.


### Aqui está uma versão organizada com todos os comandos

---

# Vault para Desenvolvimento Local – Comandos Prontos

## 1. Configuração Inicial e Subida dos Containers

```bash
# Configura o endereço do Vault e o caminho do CA
export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_CAPATH="${PWD}/certs/ca.crt"

# Sobe os containers de Vault e Consul
docker compose up -d

# Acesso às UIs
# Vault UI: https://127.0.0.1:8200/ui
# Consul UI: http://127.0.0.1:8500/ui
```

---

## 2. Login na Imagem do Vault

```bash
docker exec -it cault_vault_1 sh
```

Verifique o status do Vault:

```bash
vault status
```

---

## 3. Inicializando o Vault

```bash
vault operator init
```

> **Atenção:** anote os 5 `Unseal Keys` e o `Initial Root Token`.

---

## 4. Deslacrando o Vault (3 vezes com 3 chaves diferentes)

```bash
vault operator unseal  # insira Unseal Key 1
vault operator unseal  # insira Unseal Key 2
vault operator unseal  # insira Unseal Key 3
```

Verifique que `Sealed` agora é `false`.

---

## 5. Autenticação no Vault

```bash
vault login
# insira o Initial Root Token
```

---

## 6. Alias para facilitar o uso do Vault fora da imagem

```bash
alias vault='docker exec -it cault_vault_1 vault "$@"'
```

---

## 7. Monitorando logs do Consul

```bash
docker logs cault_consul_1 -f
```

---

## 8. Escrevendo e Lendo Secrets no Vault

```bash
# Escrevendo secret no cubbyhole
vault write -address=http://127.0.0.1:8200 cubbyhole/billion-dollars value=behind-super-secret-password

# Lendo secret
vault read cubbyhole/billion-dollars
```

---

## 9. Response Wrapping – System Backend

```bash
# Exporta variáveis do Vault
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=s.1ee2zxWvX43sAwjlcDaSGGSC

# Cria arquivo de credenciais
cat > creds.json <<EOF
{"username": "ceo", "password": "behind-super-secret-password"}
EOF

# Gera token de uso único
token=$(./tools/vault/wrap-token.sh creds.json)
echo $token

# Desembrulha o token
./tools/vault/unwrap-token.sh $token

# Tentativa de usar o token novamente falha
./tools/vault/unwrap-token.sh $token
```

---

## 10. Cubbyhole Backend

```bash
# Exporta variáveis do Vault
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=s.1ee2zxWvX43sAwjlcDaSGGSC

# Cria token de uso único para cubbyhole
token=$(./tools/vault/cubbyhole-wrap-token.sh /cubbyhole/billion-dollars)
echo $token

# Acessa secret via token
curl -s -H "X-Vault-Token: $token" -X GET $VAULT_ADDR/v1/cubbyhole/response

# Tentativa de usar o token novamente falha
curl -s -H "X-Vault-Token: $token" -X GET $VAULT_ADDR/v1/cubbyhole/response
```

---

## 11. Backup e Restore do Consul (opcional)

```bash
# Backup
consul snapshot save backups/vault-consul-backup-$(date +%Y%m%d%H%M).snap

# Restore
consul snapshot restore ./backups/vault-consul-backup-202110021214.snap
```

---

## 12. Solução de Problemas – Imagens antigas

```bash
docker rm $(docker ps -a -q)  # Remove containers parados
```


