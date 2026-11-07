# Lab: Criando um site estático no Amazon S3 com a AWS CLI

Laboratório prático em que hospedei um site estático (Café & Bakery) em um bucket do **Amazon S3**, usando apenas a **AWS CLI** a partir de uma instância **Amazon EC2** acessada pelo **AWS Systems Manager Session Manager**.

## Sumário

1. [Visão geral e objetivos](#visão-geral-e-objetivos)
2. [Conceitos e serviços](#conceitos-e-serviços)
3. [Arquitetura](#arquitetura)
4. [Passo a passo explicado](#passo-a-passo-explicado)
5. [Desafio opcional: `cp` vs `sync`](#desafio-opcional-cp-vs-sync)
6. [Problemas que encontrei e como resolvi](#problemas-que-encontrei-e-como-resolvi)
7. [Onde isso pode ser usado](#onde-isso-pode-ser-usado)
8. [Boas práticas e limitações](#boas-práticas-e-limitações)
9. [Próximos passos](#próximos-passos)
10. [O que aprendi](#o-que-aprendi)
11. [Prints](#prints)
12. [Referências](#referências)

---

## Visão geral e objetivos

O laboratório simula uma tarefa comum de quem trabalha com cloud: publicar um site na internet sem manter um servidor web.

Objetivos:

- Executar comandos da AWS CLI que usam os serviços **IAM** e **Amazon S3**.
- Implantar um site estático em um bucket do S3.
- Criar um script que usa a AWS CLI para enviar arquivos locais ao S3 e repetir a publicação.

Resultado final: um site público acessível por uma URL no formato `http://SEU-BUCKET.s3-website-us-west-2.amazonaws.com`.

---

## Conceitos e serviços

| Serviço / conceito | O que é | Papel no laboratório |
|---|---|---|
| **Amazon EC2** | Servidores virtuais na nuvem (instâncias). | A instância Amazon Linux serviu como "máquina de comando" (CLI host), de onde rodei os comandos. |
| **AWS Systems Manager Session Manager** | Acesso ao terminal de uma instância pelo navegador, sem abrir a porta 22 (SSH) e sem chaves `.pem`. | Forma de conectar na instância. A sessão abre como `ssm-user`. |
| **AWS CLI** | Ferramenta de linha de comando para controlar serviços da AWS. Já vem instalada no Amazon Linux. | Todos os comandos `aws ...` do laboratório. |
| **Amazon S3** | Armazenamento de objetos (arquivos) escalável, durável e acessado por HTTP. | Guarda os arquivos do site e os serve ao público. |
| **Bucket** | "Contêiner" de objetos no S3. O nome é **único em toda a AWS** (todas as contas e regiões). | `SEU-BUCKET`, onde o site foi hospedado. |
| **Objeto** | Cada arquivo guardado no bucket (mais seus metadados). | `index.html`, `css/styles.css` e as imagens. |
| **Hospedagem de site estático** | Recurso do S3 que serve os arquivos de um bucket como um site (HTML, CSS, JS, imagens), com um documento de índice. | Ativado com `aws s3 website`. |
| **AWS IAM** | Serviço de identidade e permissões: define **quem** pode fazer **o quê** na conta. | Criei o usuário `awsS3user` com acesso total ao S3. |
| **Política (policy)** | Documento JSON que lista permissões. As *gerenciadas pela AWS* já vêm prontas. | `AmazonS3FullAccess`, anexada ao usuário. |
| **ACL (lista de controle de acesso)** | Mecanismo antigo de permissão por objeto. | `--acl public-read` deixa cada objeto legível por qualquer pessoa. |
| **Block Public Access** | Trava de segurança do S3 que impede acesso público, mesmo com ACLs ou políticas liberando. | Precisou ser desligada no bucket para o site ficar público. |
| **Região** | Local geográfico dos data centers da AWS. | `us-west-2` (Oregon). |
| **Site estático** | Site feito só de arquivos prontos (HTML, CSS, JS, imagens), sem código rodando no servidor. | O site Café & Bakery. |

> **Estático x dinâmico:** o S3 não executa código de servidor (PHP, Node.js, Python...). Para isso seriam necessários outros serviços, como EC2, Lambda ou containers.

---

## Arquitetura

```
 Meu computador (navegador)
        │
        ├──► Console da AWS ──► ajustes do bucket (permissões, ACL)
        │
        └──► Session Manager (AWS Systems Manager)
                    │
                    ▼
            Instância EC2 (Amazon Linux)
            └─ AWS CLI + arquivos do site (~/sysops-activity-files)
                    │  aws s3 cp / sync
                    ▼
              Bucket Amazon S3  ──►  Endpoint do site (HTTP)  ──►  Visitantes
                    ▲
              IAM (usuário + política AmazonS3FullAccess)
```

---

## Passo a passo explicado

Em todos os comandos, **`SEU-BUCKET` é só o nome**, sem os sinais `< >`. Esses sinais nos tutoriais indicam "troque pelo seu valor".

### 1. Conectar na instância (Session Manager)

A sessão abre como `ssm-user`, mas os arquivos do laboratório estão na pasta do `ec2-user`. Por isso é preciso trocar de usuário:

```bash
sudo su -l ec2-user
pwd   # /home/ec2-user
```

| Parte | Significado |
|---|---|
| `sudo` | Executa o comando com privilégios de administrador. |
| `su` | Troca de usuário (*switch user*). |
| `-l` | Abre um shell de login, carregando o ambiente (home, variáveis) do novo usuário. |
| `ec2-user` | Usuário padrão do Amazon Linux. |
| `pwd` | Mostra o diretório atual, para confirmar `/home/ec2-user`. |

Se a sessão reconectar ou você abrir outra aba, o terminal volta a ser `ssm-user`: repita o comando.

### 2. Configurar a AWS CLI

```bash
aws configure
# AWS Access Key ID:     (valor fornecido pelo lab)
# AWS Secret Access Key: (valor fornecido pelo lab)
# Default region name:   us-west-2
# Default output format: json
```

- **Access Key ID + Secret Access Key:** são as credenciais que provam quem você é para a API da AWS. Ficam salvas em `~/.aws/credentials`.
- **Region:** região usada por padrão nos comandos.
- **Output format:** formato das respostas (`json`, `text`, `table`).

> As chaves do lab são temporárias, mas **nunca** publique chaves de acesso em prints, README ou repositório. Em contas reais, chave exposta deve ser desativada imediatamente.

Para conferir quem a CLI está usando e qual é o ID da conta:

```bash
aws sts get-caller-identity
```

### 3. Criar o bucket

```bash
aws s3api create-bucket \
  --bucket SEU-BUCKET \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2
```

| Parte | Significado |
|---|---|
| `aws s3api` | Comandos de **baixo nível**, que correspondem 1 a 1 às operações da API do S3. |
| `create-bucket` | Cria o bucket. |
| `--bucket` | Nome do bucket (único no mundo; só minúsculas, números e hífens). |
| `--region` | Região onde a chamada é feita. |
| `--create-bucket-configuration LocationConstraint=...` | Define a região onde o bucket será criado. Obrigatório fora da `us-east-1`. |

Resposta esperada: um JSON com `"Location"`. Erros comuns: `BucketAlreadyExists` (nome já usado por outra conta: escolha outro).

### 4. Criar um usuário IAM com acesso total ao S3

```bash
aws iam create-user --user-name awsS3user
aws iam create-login-profile --user-name awsS3user --password SUA-SENHA
```

- `create-user`: cria a identidade no IAM. O nome é único na conta.
- `create-login-profile`: permite que o usuário entre no **Console** com senha (por padrão, usuários IAM não têm essa opção).

Procurar a política de acesso total ao S3:

```bash
aws iam list-policies --query "Policies[?contains(PolicyName,'S3')]"
```

`--query` usa a sintaxe **JMESPath** para filtrar o JSON: aqui, só as políticas cujo nome contém "S3". A correta é `AmazonS3FullAccess` (cuidado com parecidas, como `AmazonS3ReadOnlyAccess`, que só lê).

Anexar a política ao usuário:

```bash
aws iam attach-user-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --user-name awsS3user
```

Conferir:

```bash
aws iam list-attached-user-policies --user-name awsS3user
```

> **Por que fazer isso?** Antes de anexar a política, o usuário recebe erro de acesso ao bucket (comportamento padrão do IAM: *tudo negado até ser permitido*). Foi uma forma prática de ver o **princípio do menor privilégio** funcionando. Em projetos reais, prefira uma política restrita a um bucket específico, em vez de acesso total.

### 5. Ajustar as permissões do bucket (no console)

No console do S3, no bucket, aba **Permissões**:

1. **Bloquear acesso público (configurações do bucket)** → Editar → desmarcar **Bloquear todo o acesso público** → Salvar.
2. **Propriedade do objeto** → Editar → **ACLs habilitadas** → marcar o reconhecimento → Salvar.

Por quê: por padrão, novos buckets bloqueiam acesso público e têm as ACLs desativadas, o que protege contra vazamentos de dados. Como este site precisa ser público e usa `--acl public-read`, as duas travas precisaram ser liberadas. Sem isso, o upload falha com erro de ACL ou o site retorna **403 Forbidden**.

### 6. Extrair os arquivos do site

```bash
cd ~/sysops-activity-files
tar xvzf static-website-v2.tar.gz
cd static-website
ls   # css  images  index.html
```

| Parte | Significado |
|---|---|
| `tar` | Empacota e desempacota arquivos. |
| `x` | Extrair (*extract*). |
| `v` | Modo verboso: mostra os arquivos extraídos. |
| `z` | Arquivo compactado com gzip (`.tar.gz`). |
| `f` | O próximo argumento é o nome do arquivo. |

### 7. Ativar o site e enviar os arquivos

```bash
# Define o index.html como página inicial (ativa a hospedagem de site estático)
aws s3 website s3://SEU-BUCKET/ --index-document index.html

# Envia todos os arquivos com leitura pública
aws s3 cp /home/ec2-user/sysops-activity-files/static-website/ s3://SEU-BUCKET/ \
  --recursive --acl public-read

# Verifica o envio
aws s3 ls SEU-BUCKET
```

| Parte | Significado |
|---|---|
| `aws s3` | Comandos de **alto nível** (mais simples), construídos sobre o `s3api`. |
| `website` | Configura a hospedagem de site estático do bucket. |
| `--index-document` | Arquivo servido quando alguém acessa a raiz do site. |
| `cp` | Copia arquivos (origem → destino). |
| `s3://SEU-BUCKET/` | Formato de endereço de um bucket na CLI. |
| `--recursive` | Inclui subpastas (`css/`, `images/`). |
| `--acl public-read` | Cada objeto enviado pode ser lido por qualquer pessoa. |
| `ls` | Lista o conteúdo do bucket. |

O site fica disponível no **Endpoint do site do bucket** (console do S3 → bucket → aba **Propriedades** → final da página):

```
http://SEU-BUCKET.s3-website-us-west-2.amazonaws.com
```

### 8. Script para repetir a publicação

```bash
cd ~
touch update-website.sh
vi update-website.sh
```

Conteúdo do `update-website.sh` (duas linhas):

```bash
#!/bin/bash
aws s3 cp /home/ec2-user/sysops-activity-files/static-website/ s3://SEU-BUCKET/ --recursive --acl public-read
```

- `#!/bin/bash` (*shebang*): diz ao sistema que o arquivo deve ser executado pelo Bash.
- `touch`: cria um arquivo vazio.
- `vi`: editor de texto do terminal. `i` entra no modo de edição; `Esc` e depois `:wq` salva e sai.

Tornar executável, alterar o site e publicar:

```bash
chmod +x update-website.sh

# Em index.html: aquamarine -> gainsboro (2x) e orange -> cornsilk
vi sysops-activity-files/static-website/index.html

./update-website.sh
```

- `chmod +x`: dá permissão de execução ao arquivo.
- `./update-website.sh`: executa o script no diretório atual.

Depois de atualizar a página no navegador (Ctrl+F5, para ignorar o cache), as novas cores aparecem. Assim, qualquer mudança local pode ser publicada com **um único comando**.

---

## Desafio opcional: `cp` vs `sync`

Troque o `aws s3 cp` por `aws s3 sync` no script:

```bash
aws s3 sync /home/ec2-user/sysops-activity-files/static-website/ s3://SEU-BUCKET/ --acl public-read
```

| Comando | Comportamento |
|---|---|
| `aws s3 cp --recursive` | Reenvia **todos** os arquivos a cada execução. |
| `aws s3 sync` | Compara origem e destino (tamanho e data de modificação) e envia **somente o que mudou**. |

Neste caso, o `sync` enviou apenas o `index.html`. Em sites maiores, isso economiza tempo, banda e requisições. O `sync` também aceita `--delete` para remover do bucket o que foi apagado localmente (use com cuidado).

---

## Problemas que encontrei e como resolvi

| Erro | Causa | Solução |
|---|---|---|
| `No such file or directory` ao usar `<nome-do-bucket>` | Deixei os sinais `< >` no comando; o bash os lê como redirecionamento de arquivo | Usar apenas o nome, sem `< >` |
| `OperationAborted` / `BucketAlreadyExists` | Nome de bucket de exemplo já em uso (nomes são globais) | Escolher um nome próprio com números aleatórios |
| `cd: ... /home/ssm-user/sysops-activity-files: No such file` | Terminal estava como `ssm-user` | `sudo su -l ec2-user` antes de entrar na pasta |
| `Unknown options: nome-do-bucket/` | Espaço depois de `s3://` | Escrever `s3://nome-do-bucket/` sem espaço |
| `NoSuchBucket` | O bucket ainda não existia nesta sessão | Criar o bucket antes do upload |
| Colei comandos dentro do arquivo `update-website.sh` | Confundi o que vai **no arquivo** com o que se **roda no terminal** | Apagar as linhas extras no `vi` (`:3,5d`) e salvar |

---

## Onde isso pode ser usado

**O que o S3 com hospedagem estática atende bem:**

- Site institucional, portfólio, blog estático e landing pages.
- Documentação de projetos (gerada por Hugo, Jekyll, MkDocs, Docusaurus...).
- Aplicações de página única (SPA) feitas em React, Vue ou Angular, depois do `build`.
- Hospedagem de arquivos públicos (imagens, CSS, JS, PDFs, downloads).
- Páginas temporárias, como "em manutenção" ou "fora do ar", para o caso de o site principal cair.

**Outros usos do S3 e da AWS CLI além do site:**

- **Backup e arquivamento** de arquivos e bancos de dados, com `aws s3 sync` em rotinas agendadas (cron).
- **Data lake:** base para análises com serviços como Athena, Glue e Redshift.
- **Armazenamento de logs** de aplicações e serviços da AWS.
- **Entrega de arquivos** de aplicações (uploads de usuários, anexos, mídia).
- **Automação em pipelines CI/CD** (GitHub Actions, Jenkins, CodePipeline): o mesmo `aws s3 sync` publica um site a cada `git push`.
- **Scripts de administração:** a CLI permite criar usuários, listar recursos e automatizar tarefas em qualquer serviço da AWS, sem abrir o console.

**Evoluções comuns deste mesmo projeto:**

| Necessidade | Serviço que resolve |
|---|---|
| HTTPS (o endpoint do S3 só serve HTTP) | **Amazon CloudFront** + certificado do **AWS Certificate Manager** |
| Domínio próprio (ex.: `meusite.com.br`) | **Amazon Route 53** (ou DNS de outro provedor) |
| Mais velocidade e menor latência no mundo todo | **CloudFront** (rede de entrega de conteúdo, CDN) |
| Publicar automaticamente a cada commit | **GitHub Actions** ou **AWS CodePipeline** |
| Manter o bucket privado e servir só pelo CDN | **CloudFront** com **Origin Access Control (OAC)** |
| Infraestrutura criada por código | **AWS CloudFormation**, **Terraform** ou **AWS CDK** |
| Backend para o site (formulários, APIs) | **Amazon API Gateway** + **AWS Lambda** |

---

## Boas práticas e limitações

**Limitações do que fiz no lab:**

- O endpoint de site do S3 usa só **HTTP**, sem HTTPS.
- O bucket ficou **totalmente público** e com ACLs, o que serve para estudo, mas não é o recomendado hoje.
- O S3 não executa código de servidor: só serve arquivos prontos.

**Boas práticas para ambientes reais:**

- Preferir **bucket policy** à ACL por objeto. A AWS recomenda manter as ACLs desativadas e controlar o acesso por políticas.
- Manter o bucket **privado** e servir pelo **CloudFront** com OAC, em vez de abrir o acesso público direto.
- Usar o princípio do **menor privilégio**: em vez de `AmazonS3FullAccess`, uma política que libere só as ações e o bucket necessários.
- Não usar chaves de acesso de longa duração; preferir **perfis de instância (IAM Roles)** ou **IAM Identity Center**.
- Nunca colocar chaves e senhas em scripts ou repositórios.
- Habilitar **versionamento** do bucket para recuperar arquivos apagados ou sobrescritos.
- Acompanhar custos (armazenamento, requisições e transferência de dados) e apagar recursos que não são mais usados.

---

## Próximos passos

- [ ] Repetir o laboratório pelo **console**, para comparar com a CLI.
- [ ] Publicar o site com **CloudFront + HTTPS**.
- [ ] Trocar a ACL por uma **bucket policy** e reativar o bloqueio de acesso público, usando só o CloudFront.
- [ ] Automatizar o deploy com **GitHub Actions**.
- [ ] Recriar a mesma infraestrutura com **Terraform** ou **CloudFormation**.

---

## O que aprendi

- Criar e configurar buckets do S3 e a hospedagem de site estático pela AWS CLI.
- Criar usuários IAM e anexar políticas gerenciadas, e como o IAM nega tudo por padrão.
- Diferença entre os comandos `s3api` (baixo nível) e `s3` (alto nível), e entre `cp` e `sync`.
- Por que o S3 bloqueia acesso público por padrão e o que é preciso liberar para publicar um site.
- Automatizar uma publicação com um script shell.
- Ler mensagens de erro da CLI para achar a causa (`NoSuchBucket`, `BucketAlreadyExists`, `Unknown options`...).
- Acessar uma instância sem SSH, usando o Session Manager.

---

## Prints

<!-- Adicione aqui os prints (sem chaves de acesso visíveis):
![Site antes](imagens/site-antes.png)
![Site depois](imagens/site-depois.png)
![Saída do update-website.sh](imagens/update-website.png)
![Hospedagem de site estático habilitada](imagens/static-hosting.png)
-->

---

## Referências

- [AWS CLI: `aws s3 sync`](https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html)
- [Hospedagem de site estático no Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [Políticas gerenciadas do IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html)
- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- Laboratório "Creating a Website on S3", AWS Training and Certification
