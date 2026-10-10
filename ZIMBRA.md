# Vandelay 1.0.11-zimbra.1 — Linux amd64

Integra focalcrest/main (453380d) com stalwartlabs/main (486a384). Preserva os três commits do focalcrest e incorpora os 12 commits do upstream. Merge resolvido em EWS, IMAP, ManageSieve e exportação de e-mails. O batching mantém o retry de serverUnavailable do upstream; serverPartialFail não é repetido para evitar duplicar operações parcialmente aplicadas.

Inclui impersonação SASL PLAIN (`--auth-proxy-user`), cookie DAV (`--auth-cookie`), memória limitada no IMAP, exportação JMAP em lotes e correção de conexão IMAP após tag mismatch. Não implementa importação de .tgz nem atualização de flags/pastas de e-mails já encontrados no destino.

## Instalação

O executável x86_64-unknown-linux-musl tem SQLite embarcado. Não exige Rust nem libc dinâmica. Precisa de certificados CA para conexões TLS.

```bash
tar -xzf vandelay-linux-amd64.tar.gz -C ./
mkdir -p "$HOME/.local/bin"
install -m 0755 vandelay "$HOME/.local/bin/vandelay"
"$HOME/.local/bin/vandelay" --version
```

## Uso

Use uma conta piloto no destino. O import lê a origem e grava somente o arquivo SQLite local.

```bash
CONTA='usuario@example.com'
ZIMBRA_HOST='zimbra.example.com'
ZIMBRA_ADMIN='admin@example.com'
unset VANDELAY_PASSWORD
vandelay import imap --url "imaps://${ZIMBRA_HOST}:993" \
  --auth-basic "$ZIMBRA_ADMIN" --auth-proxy-user "$CONTA" conta.sqlite
vandelay inspect conta.sqlite
```

A senha do administrador é solicitada. O Zimbra precisa oferecer AUTH=PLAIN e autorizar a impersonação; não existe fallback para LOGIN nesse modo.

Para contatos e calendários, obtenha um token de delegação no servidor Zimbra como usuário zimbra:

```bash
zmsoap -z -t admin DelegateAuthRequest/account=usuario@example.com @by=name
```

Use o authToken retornado, enquanto válido:

```bash
read -rsp 'Token delegado Zimbra: ' ZIMBRA_TOKEN; echo
vandelay import caldav --url "https://${ZIMBRA_HOST}/dav/${CONTA}/" \
  --auth-cookie "ZM_AUTH_TOKEN=${ZIMBRA_TOKEN}" conta.sqlite
vandelay import carddav --url "https://${ZIMBRA_HOST}/dav/${CONTA}/" \
  --auth-cookie "ZM_AUTH_TOKEN=${ZIMBRA_TOKEN}" conta.sqlite
unset ZIMBRA_TOKEN
vandelay inspect conta.sqlite
```

O argumento --auth-cookie fica visível na lista de processos. Os testes locais usam mocks; não substituem uma validação em Zimbra/Stalwart reais.

```bash
unset VANDELAY_PASSWORD
vandelay export --url https://stalwart.example.com --auth-basic admin@example.com \
  --account-name piloto@example.com --objects mailbox,email --dry-run conta.sqlite
# Depois de verificar o plano, executar sem --dry-run.
```

## Release no fork

O workflow release.yml compila, testa e publica Linux amd64 quando recebe uma tag v*-zimbra.*. Também aceita execução manual para gerar artifact, sem publicar release. Não publica nos canais npm/Homebrew do upstream.

```bash
git tag v1.0.11-zimbra.1
git push origin v1.0.11-zimbra.1
```

Sincronizações futuras devem ser feitas numa branch, com `git fetch upstream` e `git merge upstream/main`, resolvendo conflitos e repetindo os testes. Não usar reset --hard para sincronizar e apagar os patches Zimbra.
