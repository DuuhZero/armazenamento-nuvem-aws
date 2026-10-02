# Guia de execução — Atividade 3 (Armazenamento AWS) — Grupo Skyfall

Este guia é para quem vai fazer a parte prática na AWS. Os comandos são de **copiar e colar**, um bloco por vez. Os prints são para colar no Word `Relatorio_Atividade3_Armazenamento_Skyfall.docx`, cada um no lugar do `[INSERIR PRINT: ...]` com o mesmo nome.

Tempo total: cerca de 1 hora. Metade disso é esperar o banco RDS ficar pronto.

> 📸 **= tirar print.** Cada print tem a seção do Word onde ele entra.
> ✏️ **= anotar.** É uma informação para preencher nos campos amarelos do Word.

---

## Passo 0 — Preparação (5 min)

1. Entre no **console da AWS** (conta pessoal ou AWS Academy → *Start Lab* → clicar em **AWS** quando a bolinha ficar verde).
2. No canto superior direito, confira que a região é **N. Virginia (us-east-1)**.
3. Abra o **CloudShell**: ícone `>_` na barra do topo, ao lado do sino. É um terminal no navegador, já logado e com tudo instalado. **Todos os comandos deste guia são colados nele.**
   *Se o CloudShell não abrir (alguns Learner Labs bloqueiam): instale o AWS CLI no seu PC, rode `aws configure` (ou cole as credenciais de "AWS Details" do Learner Lab em `~/.aws/credentials`) e rode os mesmos comandos no Git Bash.*
4. No **seu PC** (não no CloudShell), abra https://checkip.amazonaws.com e anote o número que aparecer. É o seu IP. ✏️ (Word, seção 2.2)
5. Instale o **DBeaver** no seu PC: https://dbeaver.io/download

✏️ Anote o **ID da conta AWS**: clique no seu nome no canto superior direito e copie o *Account ID* (12 dígitos). Ele vai na capa do Word.

Cole isto no CloudShell. **Troque a senha e o IP** antes de dar Enter:

```bash
export BUCKET=fatec-dsm-skyfall
export REGION=us-east-1
export DB_ID=fatec-skyfall-rds
export DB_PASS='SkyfallSenha2026'      # troque se quiser (mín. 8 caracteres, sem @ / " ou espaço)
export MEU_IP='COLE_SEU_IP_AQUI'       # o número do checkip.amazonaws.com
export TABELA=fatec-skyfall-eventos
export AWS_DEFAULT_REGION=$REGION AWS_PAGER=""
mkdir -p ~/atv3 && cd ~/atv3
```

> ⚠️ Se o CloudShell ficar parado e reiniciar a sessão, cole esse bloco de novo antes de continuar.

---

## Passo 1 — Começar o RDS primeiro (ele demora ~10 min)

O RDS é a parte mais lenta, então ele é criado agora. Enquanto ele sobe, você faz o S3 e o DynamoDB.

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SG_ID=$(aws ec2 create-security-group --group-name fatec-skyfall-rds-sg --vpc-id $VPC_ID \
  --description "PostgreSQL somente IP do grupo" --query GroupId --output text)
echo "Security Group: $SG_ID"
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --ip-permissions "IpProtocol=tcp,FromPort=5432,ToPort=5432,IpRanges=[{CidrIp=$MEU_IP/32,Description=IP-do-grupo}]"

aws rds create-db-instance --db-instance-identifier $DB_ID \
  --engine postgres --db-instance-class db.t3.micro \
  --allocated-storage 20 --storage-type gp2 --no-multi-az \
  --publicly-accessible --vpc-security-group-ids $SG_ID \
  --master-username tutoradmin --master-user-password "$DB_PASS" \
  --db-name tutordb --backup-retention-period 1 \
  --tags Key=Projeto,Value=API-Tutor Key=Grupo,Value=skyfall \
  --query "DBInstance.{Id:DBInstanceIdentifier,Status:DBInstanceStatus}" --output table
```

Deve aparecer uma tabelinha com `Status: creating`. **Não espere.** Vá para o Passo 2.

> Se der erro em `db.t3.micro`, troque por `db.t4g.micro` no comando e rode de novo.
> Se der `InvalidGroup.Duplicate`, o SG já existe. Pegue o ID com
> `SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=fatec-skyfall-rds-sg --query 'SecurityGroups[0].GroupId' --output text)`
> e rode só o `create-db-instance`.

---

## Passo 2 — S3 (10 min)

### 2.1 Criar os arquivos de exemplo

```bash
mkdir -p publico privado v2
echo 'iVBORw0KGgoAAAANSUhEUgAAASwAAAB4BAMAAAC6O9ITAAAAGFBMVEX////n6u6yvcj/qgCcqrh+io80T2cSNFZ8vTTEAAADgUlEQVR42u2aTU/UUBSGn1s6sr2oxO0MiOvRjduiJG7RhLg1JiNbRIXfAJq4Jhr3Rg3zA8akWxITXYvC7A1Qt1JaF53B3n4MRZlp1dMNc2879Ol73vNRglqlioeFYAmWYAmWYAmWYAmWYAmWYAmWYAmWYAnWf4tlZ212AGynWlid6IffKRHMyqMCfLcyWH4nb1EqljtwWRZWUp6y4mhi+anzfhWwMrRxy8fKksYvHytTGbdsrGxh/LKxcnRxy8XKk8UvFytXFbfsAlHBeSs/Vn6ZWANCNfoopuetsdn8FrnhHX8cXzrlnX48h3MP+59ATd/U0a+81Sw2nZ72OHpGbfmkiw6AILYOv3x9oE8M4iADnYm5usCRsRO+qkAmfgJCz9g69E7EGmjrM/B84PUlix3t4pY/6sDYLOH7jKsXgS2XS/d+w3/R8/XcbT9i72Xkt+IvZEUe/inWE9hyuVvfcuFwjdoyBB8+qos3NITrqJW9d9+f9L+wD1YQ9/yF+TbhWWPlwL7w4OBrK8qw4K33y7oeTG8bntcFyungZCuaim88gGAzWn3zTMdf0z2HjbgnHnYNnrihwy6qriGG5Q0H6/ir1x9DbXV1mR2wFlsQuiSrQQBjNOM5vdcGNRJvuTCjmW+zC4BaqO/EElHROM49fw2AiVFghR44MEUvwyYbTMdaz3mU2X5gfhTeCkFpsPpFatZsPU3GEu2npkdi+cgsqq+WNhOxjqUJY3Ve3R9WJnYTahmI2mg9SkMzao3R6cv5A8SfeCvIkiq/9YRrsQexHw2zboXdtL0IMwD38x/mTLEiAwWpPa9XohKHl2jZRbHsU47WQLgZfkiZyYWdDLXcxIAzNLU0fF53j2XyPcCBbW+/DY2Milak5/y55ZuuGb0NxpemINgAlJPODrUC8LqLe2+IanEVQNWNUlCLlkzqdCJGdnMGjH6ZWM6gqzJO1jQw0+O43dtc0ADW7eTFu/0bTRRMxd9WS7Um1JX+/SfvRPpYrYZWMy2d8Xox0fdMoVRUq3mvqnEDOvKnkb8DyzmV40eGZQ+jnf+73nKqE8M4ll2dGBpBdCojloFlV0Ys0/JOVcQyseyqiJUoEE5FxEpg2RURK1lO50qfHbKrvFOFEKax7Lm8Rck9ca70CGZbeq6a/2OTNr4MNoIlWIIlWIIlWIIlWIIlWIIlWIIlWIL1b2L9BPm34z07fU0UAAAAAElFTkSuQmCC' | base64 -d > publico/logo-tutor.png
echo 'JVBERi0xLjQKMSAwIG9iago8PCAvVHlwZSAvQ2F0YWxvZyAvUGFnZXMgMiAwIFIgPj4KZW5kb2JqCjIgMCBvYmoKPDwgL1R5cGUgL1BhZ2VzIC9LaWRzIFszIDAgUl0gL0NvdW50IDEgPj4KZW5kb2JqCjMgMCBvYmoKPDwgL1R5cGUgL1BhZ2UgL1BhcmVudCAyIDAgUiAvTWVkaWFCb3ggWzAgMCA1OTUgODQyXSAvQ29udGVudHMgNCAwIFIgL1Jlc291cmNlcyA8PCAvRm9udCA8PCAvRjEgNSAwIFIgPj4gPj4gPj4KZW5kb2JqCjQgMCBvYmoKPDwgL0xlbmd0aCAxNDIgPj4Kc3RyZWFtCkJUIC9GMSAyMCBUZiA2MCA3NjAgVGQgKEd1aWEgZG8gQWx1bm8gLSBTaXN0ZW1hIFR1dG9yKSBUaiAvRjEgMTIgVGYgMCAtNDAgVGQgKE1hdGVyaWFsIGRlIGV4ZW1wbG8gaG9zcGVkYWRvIG5vIEFtYXpvbiBTMyAtIEF0aXZpZGFkZSAzLikgVGogRVQKZW5kc3RyZWFtCmVuZG9iago1IDAgb2JqCjw8IC9UeXBlIC9Gb250IC9TdWJ0eXBlIC9UeXBlMSAvQmFzZUZvbnQgL0hlbHZldGljYSA+PgplbmRvYmoKeHJlZgowIDYKMDAwMDAwMDAwMCA2NTUzNSBmIAowMDAwMDAwMDA5IDAwMDAwIG4gCjAwMDAwMDAwNTggMDAwMDAgbiAKMDAwMDAwMDExNSAwMDAwMCBuIAowMDAwMDAwMjQxIDAwMDAwIG4gCjAwMDAwMDA0MzQgMDAwMDAgbiAKdHJhaWxlcgo8PCAvU2l6ZSA2IC9Sb290IDEgMCBSID4+CnN0YXJ0eHJlZgo1MDQKJSVFT0YK' | base64 -d > publico/guia-do-aluno.pdf
printf 'Sistema Tutor - Notas da Versao\nVersao: 1.0\n- Cadastro de alunos e cursos\n- Trilhas de aprendizagem iniciais\n' > publico/notas-da-versao.txt
printf 'Sistema Tutor - Notas da Versao\nVersao: 2.0\n- Cadastro de alunos e cursos\n- Trilhas de aprendizagem iniciais\n- NOVO: eventos em tempo real (DynamoDB)\n- NOVO: materiais no Amazon S3\n' > v2/notas-da-versao.txt
cat > privado/alunos.csv <<'X'
ra,nome,email,curso,semestre
2025001,Ana Souza,ana.souza@fatec.sp.gov.br,DSM,4
2025002,Bruno Lima,bruno.lima@fatec.sp.gov.br,DSM,4
2025003,Carla Mendes,carla.mendes@fatec.sp.gov.br,ADS,3
2025004,Diego Rocha,diego.rocha@fatec.sp.gov.br,DSM,2
2025005,Elisa Prado,elisa.prado@fatec.sp.gov.br,ADS,5
X
cat > privado/config-trilhas.json <<'X'
{"versao":1,"trilhas":[{"id":"BD-01","nome":"Banco de Dados","modulos":6},{"id":"NUV-01","nome":"Computacao em Nuvem","modulos":8},{"id":"WEB-01","nome":"Desenvolvimento Web","modulos":10}],"gabarito_chave_interna":"NAO-PUBLICAR"}
X
cat > index.html <<'X'
<!DOCTYPE html><html lang="pt-BR"><head><meta charset="UTF-8"><title>Sistema Tutor - Skyfall</title>
<style>body{font-family:Arial;max-width:700px;margin:40px auto;padding:0 16px}</style></head><body>
<h1>Sistema Tutor de Aprendizagem</h1><p>Site estatico no Amazon S3 - Grupo Skyfall - FATEC DSM</p>
<img src="publico/logo-tutor.png" alt="logo">
<ul><li><a href="publico/guia-do-aluno.pdf">Guia do aluno (PDF)</a></li>
<li><a href="publico/notas-da-versao.txt">Notas da versao (TXT)</a></li>
<li><a href="privado/alunos.csv">Arquivo privado (deve dar erro 403)</a></li></ul></body></html>
X
echo '<html><body><h1>Pagina nao encontrada ou acesso negado</h1><a href="/index.html">Voltar</a></body></html>' > error.html
ls -R
```

### 2.2 Bucket, versionamento, upload e permissões

```bash
aws s3api create-bucket --bucket $BUCKET --region $REGION
aws s3api put-bucket-tagging --bucket $BUCKET --tagging "TagSet=[{Key=Projeto,Value=API-Tutor},{Key=Grupo,Value=skyfall}]"
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false

aws s3 cp publico/ s3://$BUCKET/publico/ --recursive
aws s3 cp privado/ s3://$BUCKET/privado/ --recursive
aws s3 cp index.html s3://$BUCKET/index.html --content-type "text/html; charset=utf-8"
aws s3 cp error.html s3://$BUCKET/error.html --content-type "text/html; charset=utf-8"

cat > policy.json <<X
{"Version":"2012-10-17","Statement":[{"Sid":"LeituraPublicaMateriais","Effect":"Allow","Principal":"*",
"Action":"s3:GetObject","Resource":["arn:aws:s3:::$BUCKET/publico/*","arn:aws:s3:::$BUCKET/index.html","arn:aws:s3:::$BUCKET/error.html"]}]}
X
aws s3api put-bucket-policy --bucket $BUCKET --policy file://policy.json && echo "POLICY OK"
```

> **Se o bucket der `BucketAlreadyExists`:** outra pessoa no mundo já usa esse nome. Troque para `export BUCKET=fatec-dsm-skyfall-2026`, rode o bloco de novo e use esse nome no Word (Ctrl+L → substituir `fatec-dsm-skyfall` por `fatec-dsm-skyfall-2026`).
> **Se a policy der `AccessDenied`:** vá em S3 → *Block Public Access settings for this account* (menu da esquerda) → Edit → desmarque tudo → salve, e rode o `put-bucket-policy` de novo.

### 2.3 Versão 2 do arquivo, lifecycle e site

```bash
sleep 3
aws s3 cp v2/notas-da-versao.txt s3://$BUCKET/publico/notas-da-versao.txt
aws s3api list-object-versions --bucket $BUCKET --prefix publico/notas-da-versao.txt \
  --query "Versions[].{VersionId:VersionId,IsLatest:IsLatest,LastModified:LastModified}" --output table

cat > lifecycle.json <<'X'
{"Rules":[
 {"ID":"privado-para-IA-e-Glacier","Filter":{"Prefix":"privado/"},"Status":"Enabled",
  "Transitions":[{"Days":30,"StorageClass":"STANDARD_IA"},{"Days":90,"StorageClass":"GLACIER"}]},
 {"ID":"expirar-versoes-antigas","Filter":{"Prefix":""},"Status":"Enabled",
  "NoncurrentVersionExpiration":{"NoncurrentDays":30},"AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}}]}
X
aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET --lifecycle-configuration file://lifecycle.json && echo "LIFECYCLE OK"
aws s3 website s3://$BUCKET/ --index-document index.html --error-document error.html

echo "PUBLICO: https://$BUCKET.s3.$REGION.amazonaws.com/publico/logo-tutor.png"
echo "PRIVADO: https://$BUCKET.s3.$REGION.amazonaws.com/privado/alunos.csv"
echo "SITE:    http://$BUCKET.s3-website-$REGION.amazonaws.com"
```

✏️ A tabela que aparece tem **2 VersionIds** com as datas. Copie para a tabela da seção **1.3** do Word. O que tem `IsLatest = True` é a Versão 2.0.

### 📸 Prints do S3 (console → pesquise "S3" → clique no bucket `fatec-dsm-skyfall`)

| Print | O que mostrar | Vai na seção do Word |
|---|---|---|
| S3-1 | Lista de buckets mostrando `fatec-dsm-skyfall` e a região *US East (N. Virginia)* | 1.1 |
| S3-2 | Aba **Objects** do bucket (pastas `publico/`, `privado/`, `index.html`, `error.html`). Entre também em `publico/` e em `privado/` se quiser mostrar os arquivos | 1.2 (listagem) |
| S3-3 | Navegador abrindo o link **PUBLICO** (aparece a imagem) **e** o link **PRIVADO** (aparece `AccessDenied`). Pode ser um print de cada, colados juntos | 1.2 (URL pública) |
| S3-4 | Entre em `publico/` → clique em `notas-da-versao.txt` → aba **Versions**: as 2 versões com data. *(Ou na lista de objetos, ligue o botão "Show versions".)* | 1.3 |
| S3-5 | Aba **Management** → *Lifecycle rules* com as 2 regras | 1.4 |
| S3-6 | Aba **Properties** → role até o fim → *Static website hosting* (Enabled + link) **e** o site aberto no navegador (link **SITE**) | 1.5 |

---

## Passo 3 — DynamoDB (10 min)

### 3.1 Criar a tabela com GSI

```bash
aws dynamodb create-table --table-name $TABELA \
  --attribute-definitions AttributeName=usuarioId,AttributeType=S AttributeName=dataHoraEvento,AttributeType=S AttributeName=cursoId,AttributeType=S \
  --key-schema AttributeName=usuarioId,KeyType=HASH AttributeName=dataHoraEvento,KeyType=RANGE \
  --global-secondary-indexes '[{"IndexName":"gsi-curso-data","KeySchema":[{"AttributeName":"cursoId","KeyType":"HASH"},{"AttributeName":"dataHoraEvento","KeyType":"RANGE"}],"Projection":{"ProjectionType":"ALL"}}]' \
  --billing-mode PAY_PER_REQUEST --tags Key=Projeto,Value=API-Tutor Key=Grupo,Value=skyfall \
  --query "TableDescription.TableStatus"
aws dynamodb wait table-exists --table-name $TABELA && echo "TABELA PRONTA"
aws dynamodb update-time-to-live --table-name $TABELA --time-to-live-specification "Enabled=true,AttributeName=expiraEm"
```

### 3.2 Inserir os 18 eventos + 1 via put-item

```bash
cat > itens.json <<'X'
{"fatec-skyfall-eventos":[
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025001"},"dataHoraEvento":{"S":"2026-09-28T08:01:12Z#evt-0001"},"eventoId":{"S":"evt-0001"},"tipoEvento":{"S":"LOGIN"},"expiraEm":{"N":"1853654472"},"dispositivo":{"S":"web"},"ip":{"S":"177.10.20.30"},"navegador":{"S":"Chrome 129"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025001"},"dataHoraEvento":{"S":"2026-09-28T08:03:40Z#evt-0002"},"eventoId":{"S":"evt-0002"},"tipoEvento":{"S":"ACESSO_AULA"},"expiraEm":{"N":"1853654620"},"cursoId":{"S":"BD-01"},"aulaId":{"S":"BD-01-A05"},"dispositivo":{"S":"web"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025001"},"dataHoraEvento":{"S":"2026-09-28T08:25:02Z#evt-0003"},"eventoId":{"S":"evt-0003"},"tipoEvento":{"S":"EXERCICIO_RESPONDIDO"},"expiraEm":{"N":"1853655902"},"cursoId":{"S":"BD-01"},"exercicioId":{"S":"EX-0507"},"acertou":{"BOOL":true},"tentativas":{"N":"1"},"tempoSeg":{"N":"184"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025001"},"dataHoraEvento":{"S":"2026-09-29T19:10:55Z#evt-0004"},"eventoId":{"S":"evt-0004"},"tipoEvento":{"S":"PERGUNTA_TUTOR"},"expiraEm":{"N":"1853781055"},"cursoId":{"S":"NUV-01"},"pergunta":{"S":"Qual a diferença entre S3 Standard e Glacier?"},"tokens":{"N":"412"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025001"},"dataHoraEvento":{"S":"2026-09-29T19:40:00Z#evt-0005"},"eventoId":{"S":"evt-0005"},"tipoEvento":{"S":"LOGOUT"},"expiraEm":{"N":"1853782800"},"dispositivo":{"S":"web"},"duracaoSessaoSeg":{"N":"1805"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025002"},"dataHoraEvento":{"S":"2026-09-28T10:00:00Z#evt-0006"},"eventoId":{"S":"evt-0006"},"tipoEvento":{"S":"LOGIN"},"expiraEm":{"N":"1853661600"},"dispositivo":{"S":"android"},"versaoApp":{"S":"2.1.0"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025002"},"dataHoraEvento":{"S":"2026-09-28T10:02:31Z#evt-0007"},"eventoId":{"S":"evt-0007"},"tipoEvento":{"S":"VIDEO_ASSISTIDO"},"expiraEm":{"N":"1853661751"},"cursoId":{"S":"WEB-01"},"aulaId":{"S":"WEB-01-A02"},"percentual":{"N":"87"},"velocidade":{"N":"1.5"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025002"},"dataHoraEvento":{"S":"2026-09-30T21:15:09Z#evt-0008"},"eventoId":{"S":"evt-0008"},"tipoEvento":{"S":"DOWNLOAD_MATERIAL"},"expiraEm":{"N":"1853874909"},"cursoId":{"S":"WEB-01"},"arquivoS3":{"S":"publico/guia-do-aluno.pdf"},"tamanhoKB":{"N":"2"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025003"},"dataHoraEvento":{"S":"2026-09-29T14:20:00Z#evt-0009"},"eventoId":{"S":"evt-0009"},"tipoEvento":{"S":"ACESSO_AULA"},"expiraEm":{"N":"1853763600"},"cursoId":{"S":"NUV-01"},"aulaId":{"S":"NUV-01-A07"},"dispositivo":{"S":"ios"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025003"},"dataHoraEvento":{"S":"2026-09-29T14:48:13Z#evt-0010"},"eventoId":{"S":"evt-0010"},"tipoEvento":{"S":"EXERCICIO_RESPONDIDO"},"expiraEm":{"N":"1853765293"},"cursoId":{"S":"NUV-01"},"exercicioId":{"S":"EX-0712"},"acertou":{"BOOL":false},"tentativas":{"N":"3"},"dicasUsadas":{"L":[{"S":"dica-1"},{"S":"dica-2"}]}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025003"},"dataHoraEvento":{"S":"2026-10-01T09:05:44Z#evt-0011"},"eventoId":{"S":"evt-0011"},"tipoEvento":{"S":"CONQUISTA_DESBLOQUEADA"},"expiraEm":{"N":"1853917544"},"cursoId":{"S":"NUV-01"},"medalha":{"S":"Explorador da Nuvem"},"pontos":{"N":"50"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025005"},"dataHoraEvento":{"S":"2026-09-30T07:30:00Z#evt-0012"},"eventoId":{"S":"evt-0012"},"tipoEvento":{"S":"LOGIN"},"expiraEm":{"N":"1853825400"},"dispositivo":{"S":"web"},"mfa":{"BOOL":true}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025005"},"dataHoraEvento":{"S":"2026-09-30T07:31:18Z#evt-0013"},"eventoId":{"S":"evt-0013"},"tipoEvento":{"S":"ACESSO_AULA"},"expiraEm":{"N":"1853825478"},"cursoId":{"S":"BD-01"},"aulaId":{"S":"BD-01-A06"},"dispositivo":{"S":"web"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025005"},"dataHoraEvento":{"S":"2026-09-30T07:59:47Z#evt-0014"},"eventoId":{"S":"evt-0014"},"tipoEvento":{"S":"EXERCICIO_RESPONDIDO"},"expiraEm":{"N":"1853827187"},"cursoId":{"S":"BD-01"},"exercicioId":{"S":"EX-0611"},"acertou":{"BOOL":true},"tentativas":{"N":"1"},"tempoSeg":{"N":"95"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025005"},"dataHoraEvento":{"S":"2026-10-01T18:12:03Z#evt-0015"},"eventoId":{"S":"evt-0015"},"tipoEvento":{"S":"PERGUNTA_TUTOR"},"expiraEm":{"N":"1853950323"},"cursoId":{"S":"BD-01"},"pergunta":{"S":"Quando usar JOIN e quando usar subquery?"},"tokens":{"N":"388"},"avaliacaoResposta":{"N":"5"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025008"},"dataHoraEvento":{"S":"2026-09-28T23:55:21Z#evt-0016"},"eventoId":{"S":"evt-0016"},"tipoEvento":{"S":"ERRO_APP"},"expiraEm":{"N":"1853711721"},"dispositivo":{"S":"android"},"codigoErro":{"S":"NET_TIMEOUT"},"detalhe":{"M":{"tela":{"S":"player"},"versaoApp":{"S":"2.0.9"}}}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025008"},"dataHoraEvento":{"S":"2026-09-29T00:02:10Z#evt-0017"},"eventoId":{"S":"evt-0017"},"tipoEvento":{"S":"ACESSO_AULA"},"expiraEm":{"N":"1853712130"},"cursoId":{"S":"IA-01"},"aulaId":{"S":"IA-01-A03"},"dispositivo":{"S":"android"}}}},
  {"PutRequest":{"Item":{"usuarioId":{"S":"aluno-2025010"},"dataHoraEvento":{"S":"2026-10-01T13:45:00Z#evt-0018"},"eventoId":{"S":"evt-0018"},"tipoEvento":{"S":"VIDEO_ASSISTIDO"},"expiraEm":{"N":"1853934300"},"cursoId":{"S":"BD-01"},"aulaId":{"S":"BD-01-A01"},"percentual":{"N":"100"},"velocidade":{"N":"1.0"}}}}
]}
X
aws dynamodb batch-write-item --request-items file://itens.json

aws dynamodb put-item --table-name $TABELA --return-consumed-capacity TOTAL --item "{
 \"usuarioId\":{\"S\":\"aluno-2025010\"},\"dataHoraEvento\":{\"S\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)#evt-cli-01\"},
 \"eventoId\":{\"S\":\"evt-cli-01\"},\"tipoEvento\":{\"S\":\"LOGIN\"},\"dispositivo\":{\"S\":\"cli\"}}"

aws dynamodb scan --table-name $TABELA --select COUNT
```

O `batch-write-item` deve devolver `"UnprocessedItems": {}` e o scan deve mostrar `"Count": 19`.

### 3.3 Queries (rodar as 5, uma de cada vez)

```bash
echo "Q1 - linha do tempo de um aluno (Partition Key)"
aws dynamodb query --table-name $TABELA --key-condition-expression "usuarioId = :u" \
  --expression-attribute-values '{":u":{"S":"aluno-2025001"}}' \
  --projection-expression "dataHoraEvento, tipoEvento" --output table
```
```bash
echo "Q2 - eventos do aluno em 30/09 (PK + intervalo na Sort Key)"
aws dynamodb query --table-name $TABELA \
  --key-condition-expression "usuarioId = :u AND dataHoraEvento BETWEEN :ini AND :fim" \
  --expression-attribute-values '{":u":{"S":"aluno-2025005"},":ini":{"S":"2026-09-30T00:00:00Z"},":fim":{"S":"2026-09-30T23:59:59Z~"}}' \
  --projection-expression "dataHoraEvento, tipoEvento" --output table
```
```bash
echo "Q3 - ultimo evento do aluno"
aws dynamodb query --table-name $TABELA --key-condition-expression "usuarioId = :u" \
  --expression-attribute-values '{":u":{"S":"aluno-2025003"}}' --no-scan-index-forward --limit 1 \
  --projection-expression "dataHoraEvento, tipoEvento" --output table
```
```bash
echo "Q4 - todos os eventos do curso BD-01 (GSI)"
aws dynamodb query --table-name $TABELA --index-name gsi-curso-data \
  --key-condition-expression "cursoId = :c" --expression-attribute-values '{":c":{"S":"BD-01"}}' \
  --projection-expression "cursoId, dataHoraEvento, usuarioId, tipoEvento" --output table
```
```bash
echo "Q5 - exercicios respondidos no curso NUV-01 (GSI + filtro)"
aws dynamodb query --table-name $TABELA --index-name gsi-curso-data \
  --key-condition-expression "cursoId = :c" --filter-expression "tipoEvento = :t" \
  --expression-attribute-values '{":c":{"S":"NUV-01"},":t":{"S":"EXERCICIO_RESPONDIDO"}}' \
  --projection-expression "cursoId, dataHoraEvento, usuarioId, tipoEvento, acertou" --output table
```

Resultados esperados: Q1 = 5 eventos · Q2 = 3 · Q3 = CONQUISTA_DESBLOQUEADA · Q4 = 6 · Q5 = 1.

### 📸 Prints do DynamoDB (tire ANTES do teste de carga do Passo 4)

| Print | O que mostrar | Vai na seção do Word |
|---|---|---|
| DDB-1 | Console → DynamoDB → *Tables* → `fatec-skyfall-eventos` → aba **Overview** (Partition key `usuarioId`, Sort key `dataHoraEvento`) + aba **Indexes** (`gsi-curso-data`) | 3.1 |
| DDB-2 | Botão **Explore table items** → lista dos 19 itens | 3.2 |
| DDB-3 | Terminal do CloudShell com a **Q4** (comando + resultado) | 3.3 |
| DDB-4 | Terminal do CloudShell com o **put-item** (mostra `CapacityUnits: 1.0`) e a **Q1** ou **Q2** | 3.4 |

✏️ Na seção 3.4, troque `[nome do integrante]` e `[data/hora]` pelo seu nome e pelo horário em que rodou.

---

## Passo 4 — Teste de carga do DynamoDB (5 min)

Este teste comprova o requisito de "altíssimo volume de escrita". Ele grava 20.000 eventos o mais rápido possível e mede a vazão. Custa menos de US$ 0,05, e os eventos se apagam sozinhos em 1 dia.

```bash
cat > carga.py <<'X'
import boto3, uuid, random, time, json, threading, statistics
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime, timedelta, timezone
from botocore.config import Config
T="fatec-skyfall-eventos"; TOTAL=20000; THREADS=16
ddb=boto3.client("dynamodb",region_name="us-east-1",config=Config(max_pool_connections=20,retries={"max_attempts":10,"mode":"adaptive"}))
TIPOS=["LOGIN","ACESSO_AULA","VIDEO_ASSISTIDO","EXERCICIO_RESPONDIDO","PERGUNTA_TUTOR","LOGOUT"]; CURSOS=["BD-01","NUV-01","WEB-01","IA-01"]
def ev():
    a=datetime.now(timezone.utc); e=uuid.uuid4().hex[:12]; t=random.choice(TIPOS)
    it={"usuarioId":{"S":f"carga-{random.randint(1,5000):05d}"},"dataHoraEvento":{"S":a.isoformat(timespec="milliseconds").replace("+00:00","Z")+"#"+e},
        "eventoId":{"S":e},"tipoEvento":{"S":t},"origem":{"S":"teste-carga"},"expiraEm":{"N":str(int((a+timedelta(days=1)).timestamp()))}}
    if t not in("LOGIN","LOGOUT"): it["cursoId"]={"S":random.choice(CURSOS)}
    return {"PutRequest":{"Item":it}}
lotes=[[ev() for _ in range(25)] for _ in range(TOTAL//25)]
lat=[]; st={"ok":0,"reenvio":0,"perdido":0}; lk=threading.Lock()
def envia(l):
    for tent in range(8):
        t0=time.perf_counter(); r=ddb.batch_write_item(RequestItems={T:l}); ms=(time.perf_counter()-t0)*1000
        np=r.get("UnprocessedItems",{}).get(T,[])
        with lk: lat.append(ms); st["ok"]+=len(l)-len(np); st["reenvio"]+=len(np)
        if not np: return
        l=np; time.sleep(0.05*2**tent)
    with lk: st["perdido"]+=len(l)
ini=time.perf_counter()
with ThreadPoolExecutor(THREADS) as x: list(x.map(envia,lotes))
d=time.perf_counter()-ini; lat.sort(); p=lambda q:round(lat[int(len(lat)*q)-1],1)
v=st["ok"]/d
print(json.dumps({"eventos_gravados":st["ok"],"eventos_perdidos":st["perdido"],"reenviados_throttling":st["reenvio"],
 "duracao_s":round(d,2),"vazao_eventos_por_s":round(v,1),"latencia_lote_ms":{"p50":p(.5),"p95":p(.95),"p99":p(.99)},
 "RNF01_atendido":v>=1000 and st["perdido"]==0 and p(.95)<=500},indent=2))
X
python3 carga.py
```

Leva uns 10 a 30 segundos e imprime o resultado.

✏️ Copie os números para a tabela da seção **3.5** do Word: gravados, perdidos, duração, vazão, p50/p95/p99, reenviados e se o RNF foi atendido (true = Sim). No campo "executado a partir de", escreva **AWS CloudShell**.

✏️ No parágrafo **"Análise"** da seção 3.5, apague o texto amarelo e escreva uma frase com o resultado. Por exemplo: *"A tabela gravou 20.000 eventos a cerca de X eventos/s, sem perdas e sem throttling, superando a meta do RNF-01."* (Se houve reenvios, cite quantos.)

### 📸 Prints

| Print | O que mostrar | Vai na seção do Word |
|---|---|---|
| CARGA-1 | Terminal com o resultado JSON do `carga.py`. Esse print substitui o "gráfico" | 3.5 (1º print) |
| CARGA-2 | Espere ~5 min → DynamoDB → tabela → aba **Monitor** → gráfico *Write usage* mostrando o pico | 3.5 (2º print) |

---

## Passo 5 — Voltar ao RDS (15 min)

### 5.1 Ver se o banco ficou pronto

```bash
aws rds wait db-instance-available --db-instance-identifier $DB_ID && \
aws rds describe-db-instances --db-instance-identifier $DB_ID --query "DBInstances[0].{Status:DBInstanceStatus,Versao:EngineVersion,Endpoint:Endpoint.Address}" --output table
```

Quando terminar, mostra `Status: available` e o **Endpoint** (algo como `fatec-skyfall-rds.xxxx.us-east-1.rds.amazonaws.com`). Copie o endpoint.

✏️ Anote a **versão** do PostgreSQL (ex.: 17.4) → seção **2.1** do Word.

### 5.2 Conectar pelo DBeaver (no seu PC)

1. DBeaver → **Nova conexão** (ícone de tomada) → **PostgreSQL** → Avançar.
2. Host: *o endpoint* · Port: `5432` · Database: `tutordb` · Username: `tutoradmin` · Password: *a senha do Passo 0*.
3. Se ele pedir para baixar o driver, aceite.
4. Clique em **Test Connection** → deve dar "Connected".

> **Se der timeout:** você está numa rede com outro IP, ou a rede (ex.: da faculdade) bloqueia a porta 5432. Veja seu IP de novo em checkip.amazonaws.com e libere no CloudShell:
> `aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 5432 --cidr NOVO_IP/32`
> Se ainda assim não conectar, use o 4G do celular (roteador) e libere esse IP.

### 5.3 Criar tabelas e dados

No DBeaver: botão direito na conexão → **SQL Editor** → **New SQL script**. Cole tudo abaixo e rode com **Alt+X** (executa o script inteiro):

```sql
DROP TABLE IF EXISTS matricula; DROP TABLE IF EXISTS aluno; DROP TABLE IF EXISTS curso;

CREATE TABLE curso (
    id SERIAL PRIMARY KEY, codigo VARCHAR(10) NOT NULL UNIQUE,
    nome VARCHAR(100) NOT NULL, carga_horaria INTEGER NOT NULL CHECK (carga_horaria > 0));
CREATE TABLE aluno (
    id SERIAL PRIMARY KEY, ra VARCHAR(13) NOT NULL UNIQUE, nome VARCHAR(100) NOT NULL,
    email VARCHAR(120) NOT NULL UNIQUE, semestre SMALLINT NOT NULL CHECK (semestre BETWEEN 1 AND 6),
    criado_em TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP);
CREATE TABLE matricula (
    id SERIAL PRIMARY KEY,
    aluno_id INTEGER NOT NULL REFERENCES aluno(id) ON DELETE CASCADE,
    curso_id INTEGER NOT NULL REFERENCES curso(id),
    progresso NUMERIC(5,2) NOT NULL DEFAULT 0 CHECK (progresso BETWEEN 0 AND 100),
    nota NUMERIC(4,2) CHECK (nota BETWEEN 0 AND 10), UNIQUE (aluno_id, curso_id));
CREATE INDEX idx_matricula_curso ON matricula(curso_id);

INSERT INTO curso (codigo, nome, carga_horaria) VALUES
 ('BD-01','Banco de Dados',80),('NUV-01','Computação em Nuvem',80),
 ('WEB-01','Desenvolvimento Web',120),('IA-01','Introdução à IA',60);
INSERT INTO aluno (ra, nome, email, semestre) VALUES
 ('2025001','Ana Souza','ana.souza@fatec.sp.gov.br',4),('2025002','Bruno Lima','bruno.lima@fatec.sp.gov.br',4),
 ('2025003','Carla Mendes','carla.mendes@fatec.sp.gov.br',3),('2025004','Diego Rocha','diego.rocha@fatec.sp.gov.br',2),
 ('2025005','Elisa Prado','elisa.prado@fatec.sp.gov.br',5),('2025006','Felipe Nunes','felipe.nunes@fatec.sp.gov.br',4),
 ('2025007','Gabriela Torres','gabriela.torres@fatec.sp.gov.br',1),('2025008','Henrique Alves','henrique.alves@fatec.sp.gov.br',6),
 ('2025009','Isabela Castro','isabela.castro@fatec.sp.gov.br',3),('2025010','João Pereira','joao.pereira@fatec.sp.gov.br',4),
 ('2025011','Karen Oliveira','karen.oliveira@fatec.sp.gov.br',2),('2025012','Lucas Martins','lucas.martins@fatec.sp.gov.br',5);
INSERT INTO matricula (aluno_id, curso_id, progresso, nota) VALUES
 (1,1,100,9.5),(1,2,75,8),(2,1,90,7),(2,3,40,NULL),(3,2,100,6.5),(3,4,20,NULL),(4,3,55,5.5),
 (5,1,100,10),(5,2,80,9),(6,2,65,7.5),(6,3,100,8.5),(7,4,10,NULL),(8,1,100,6),(8,4,95,9),
 (9,3,70,7),(10,2,100,8.8),(10,1,50,4.5),(11,4,30,NULL),(12,3,85,9.2),(12,2,60,6.8);
```

### 5.4 As 3 queries

Abra um **novo** SQL script e rode **uma de cada vez**: clique dentro da query e aperte **Ctrl+Enter**. Tire um print de cada resultado.

```sql
-- Q1: SELECT com filtro
SELECT ra, nome, email FROM aluno WHERE semestre = 4 ORDER BY nome;
```
```sql
-- Q2: JOIN
SELECT a.nome AS aluno, c.nome AS curso, m.nota
FROM matricula m JOIN aluno a ON a.id = m.aluno_id JOIN curso c ON c.id = m.curso_id
WHERE m.progresso = 100 ORDER BY m.nota DESC;
```
```sql
-- Q3: GROUP BY
SELECT c.nome AS curso, COUNT(m.id) AS matriculados,
       ROUND(AVG(m.progresso),1) AS progresso_medio, ROUND(AVG(m.nota),2) AS nota_media
FROM curso c LEFT JOIN matricula m ON m.curso_id = c.id
GROUP BY c.nome ORDER BY matriculados DESC, c.nome;
```

Resultados esperados: Q1 = 4 linhas · Q2 = 6 linhas (Elisa no topo com 10) · Q3 = 4 linhas (Computação em Nuvem com 6). Iguais às tabelas que já estão no Word.

### 5.5 Snapshot

```bash
SNAP=fatec-skyfall-rds-snap-$(date +%Y%m%d-%H%M)
aws rds create-db-snapshot --db-instance-identifier $DB_ID --db-snapshot-identifier $SNAP --query "DBSnapshot.Status"
echo "Aguardando snapshot (3-5 min)..."; aws rds wait db-snapshot-available --db-snapshot-identifier $SNAP
aws rds describe-db-snapshots --db-snapshot-identifier $SNAP --query "DBSnapshots[0].{Id:DBSnapshotIdentifier,Status:Status,Criado:SnapshotCreateTime}" --output table
```

✏️ Anote o **Id** do snapshot → seção **2.4** do Word.

### 📸 Prints do RDS

| Print | O que mostrar | Vai na seção do Word |
|---|---|---|
| RDS-1 | Console → RDS → *Databases* → `fatec-skyfall-rds` → status **Available** + aba *Connectivity & security* (endpoint visível) | 2.1 |
| RDS-2 | Na mesma aba, clique no Security Group → *Inbound rules*: só PostgreSQL 5432 com `SEU_IP/32` | 2.2 (1º print) |
| RDS-3 | DBeaver com a janela "Connected" do Test Connection (ou a conexão aberta mostrando o host = endpoint) | 2.2 (2º print) |
| RDS-4, 5, 6 | DBeaver com o resultado de Q1, Q2 e Q3 (um print por query) | 2.3 |
| RDS-7 | Console → RDS → **Snapshots** → aba *Manual*: snapshot com status *Available* e data | 2.4 |

---

## Passo 6 — Fechar o Word (10 min)

Procure tudo que está **grifado em amarelo** e resolva:

| Onde | O que fazer |
|---|---|
| Capa — Conta AWS | ID da conta (12 dígitos) |
| Capa — Link do repositório | Link do GitHub do grupo. Se não tiver, apague a linha |
| 1.3 — tabela de versões | os 2 VersionIds e as datas |
| 2.1 — versão | versão do PostgreSQL |
| 2.2 — IP / nome | seu IP e seu nome |
| 2.4 — identificador | Id do snapshot |
| 3.4 — nome / data | seu nome e o horário |
| 3.5 — tabela + análise | números do teste de carga + 1 frase |
| Todos os `[INSERIR PRINT: ...]` | apague o texto e cole o print no lugar |
| **Seção 5 (Versão 2 — API)** | é a entrega **final** (sprints), **não** é para hoje. Apague os 2 prints dessa seção ou deixe a seção inteira para a próxima entrega |

Depois:
1. Selecione tudo (Ctrl+A) → ferramenta de realce → **Sem cor**, para garantir que não sobrou amarelo.
2. **Arquivo → Salvar como → PDF** → entregar.

---

## Passo 7 — Depois da nota: apagar tudo (para não cobrar)

O RDS cobra por hora mesmo parado. Quando a atividade for avaliada, cole no CloudShell (com o bloco do Passo 0 já colado):

```bash
aws dynamodb delete-table --table-name $TABELA --query TableDescription.TableStatus
aws rds delete-db-instance --db-instance-identifier $DB_ID --skip-final-snapshot --delete-automated-backups --query DBInstance.DBInstanceStatus
for s in $(aws rds describe-db-snapshots --db-instance-identifier $DB_ID --snapshot-type manual --query 'DBSnapshots[].DBSnapshotIdentifier' --output text); do aws rds delete-db-snapshot --db-snapshot-identifier $s >/dev/null; done
aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json)" 2>/dev/null
aws s3api delete-bucket --bucket $BUCKET && echo "bucket apagado"
aws rds wait db-instance-deleted --db-instance-identifier $DB_ID
aws ec2 delete-security-group --group-id $(aws ec2 describe-security-groups --filters Name=group-name,Values=fatec-skyfall-rds-sg --query 'SecurityGroups[0].GroupId' --output text) && echo "TUDO APAGADO"
```

Se for AWS Academy Learner Lab, os créditos são do lab, mas apague mesmo assim para não gastar.
