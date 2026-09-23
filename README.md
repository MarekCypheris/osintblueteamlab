# Segurança em bancos de dados MySQL

Este guia reúne práticas para proteger contas, conexões e dados em ambientes MySQL.

> Os exemplos usam valores fictícios. Nunca publique senhas, tokens ou chaves reais no repositório.

## 1. Use contas individuais e privilégios mínimos

Evite usar `root` em aplicações. Crie uma conta específica para cada serviço e conceda somente as permissões necessárias.

```sql
CREATE USER 'app_user'@'localhost'
IDENTIFIED BY 'SUBSTITUA_POR_UMA_SENHA_FORTE';

GRANT SELECT, INSERT, UPDATE, DELETE
ON aplicacao.*
TO 'app_user'@'localhost';
```

Nesse exemplo, a aplicação pode consultar e alterar registros, mas não recebe permissões para administrar usuários ou criar e remover tabelas.

Confira os privilégios:

```sql
SHOW GRANTS FOR 'app_user'@'localhost';
```

## 2. Proteja as credenciais

- Use senhas longas, exclusivas e geradas aleatoriamente.
- Armazene segredos em um gerenciador de segredos ou em configuração protegida.
- Não coloque senhas no código, em arquivos públicos ou em exemplos de documentação.
- Evite passar a senha diretamente na linha de comando.

Para solicitar a senha de forma interativa:

```bash
mysql -h 127.0.0.1 -u app_user -p
```

Para alterar a senha de uma conta:

```sql
ALTER USER 'app_user'@'localhost'
IDENTIFIED BY 'SUBSTITUA_POR_UMA_NOVA_SENHA_FORTE';
```

## 3. Restrinja o acesso pela rede

O banco deve aceitar conexões apenas das máquinas que precisam acessá-lo.

- Evite expor a porta `3306` diretamente à internet.
- Restrinja as origens no firewall.
- Configure o endereço de escuta conforme a arquitetura.
- Evite contas com origem `%` quando for possível definir uma origem específica.

Se o banco e a aplicação estão na mesma máquina, uma configuração possível é:

```ini
[mysqld]
bind-address = 127.0.0.1
```

Essa configuração impede conexões TCP remotas. Em ambientes com aplicação e banco separados, use o endereço privado adequado e regras de firewall.

## 4. Proteja conexões com TLS

Para conexões remotas, configure TLS no servidor e valide seu certificado no cliente.

Exemplo de conexão com validação da autoridade certificadora e do nome do servidor:

```bash
mysql \
  -h  db_intranet_3.0 \
  -u app_user \
  -p \
  --ssl-mode=VERIFY_IDENTITY \
  --ssl-ca=/caminho/ca.pem
```

Também é possível exigir conexão criptografada para uma conta:

```sql
ALTER USER 'app_user'@'10.0.0.20' REQUIRE SSL;
```

O usuário e a origem devem corresponder à conta existente. A exigência de TLS na conta não substitui a validação do certificado pelo cliente.

## 5. Previna SQL Injection

Nunca monte consultas concatenando entradas fornecidas pelo usuário. Use consultas parametrizadas.

Exemplo em PHP com PDO:

```php
$stmt = $pdo->prepare(
    'SELECT id, nome FROM usuarios WHERE email = :email'
);

$stmt->execute([
    'email' => $emailInformado
]);

$usuario = $stmt->fetch();
```

Parâmetros representam valores. Nomes de tabelas, colunas e opções de ordenação devem ser escolhidos por uma lista explícita de valores permitidos.

## 6. Armazene senhas da aplicação como hashes

As senhas dos usuários da aplicação não devem ser armazenadas em texto puro nem com criptografia reversível.

Em PHP:

```php
$hash = password_hash($senha, PASSWORD_DEFAULT);
```

Para validar:

```php
if (password_verify($senhaInformada, $hashArmazenado)) {
    // Autenticação válida.
}
```

Use uma coluna com tamanho adequado, como `VARCHAR(255)`, para acomodar mudanças no formato do hash.

## 7. Mantenha backups protegidos

- Automatize os backups.
- Restrinja quem pode acessar os arquivos.
- Proteja as cópias com criptografia.
- Mantenha cópias separadas do servidor principal.
- Teste a restauração periodicamente.
- Defina o período de retenção conforme a necessidade dos dados.

Um backup que nunca foi restaurado em teste não oferece garantia de recuperação.

## 8. Monitore e atualize

Revise periodicamente:

- Contas e permissões.
- Falhas de autenticação.
- Alterações administrativas.
- Conexões de origens inesperadas.
- Consultas lentas e comportamentos fora do padrão.
- Atualizações de segurança da versão utilizada.

Proteja também os logs: eles podem conter consultas, dados pessoais ou informações operacionais sensíveis. Evite habilitar registros detalhados indiscriminadamente em produção.

## 9. Evite vazamentos pelo GitHub

Inclua arquivos locais de configuração no `.gitignore`:

```gitignore
.env
.env.*
!.env.example
config.local.php
backups/
*.sql
*.sql.gz
```

Adapte essas regras ao projeto: arquivos SQL de migração podem precisar de versionamento.

O `.gitignore` não remove arquivos que já foram versionados e não apaga conteúdos do histórico.

Se uma credencial foi publicada:

1. Revogue ou altere a credencial imediatamente.
2. Atualize os serviços que dependem dela.
3. Investigue possíveis acessos indevidos.
4. Remova o segredo do código atual.
5. Avalie a limpeza do histórico e das demais cópias.

Excluir apenas o arquivo ou criar outro commit não torna a credencial segura novamente.

## Checklist de revisão

- [ ] A aplicação não usa a conta `root`.
- [ ] Cada conta possui somente os privilégios necessários.
- [ ] As conexões são limitadas a origens autorizadas.
- [ ] Conexões remotas usam TLS com validação do certificado.
- [ ] As consultas usam parâmetros.
- [ ] As senhas da aplicação são armazenadas com hashing apropriado.
- [ ] Os backups são protegidos e têm restauração testada.
- [ ] Logs e atualizações são revisados.
- [ ] Não há credenciais reais no repositório ou na documentação.
