# Contexto: o que mudou no projeto hithub-ficha-cadastral

Cole isto na sua sessão do Claude que roda no computador, antes de pedir a
próxima alteração.

---

## O que mudou na forma de publicar (o mais importante)

O projeto `C:\Users\patro\OneDrive\Documentos\Claude Code\hithub-ficha-cadastral`
**deixou de ser publicado por `netlify deploy` pela linha de comando.**

Agora ele está num repositório GitHub privado e a Netlify publica sozinha a
cada push:

- **Repositório:** https://github.com/hithubbrasil-spec/hithub-ficha-cadastral (privado)
- **Branch de produção:** `master`
- **Projeto Netlify:** `cadastrodigitalhithub` (site id `5e6198f0-4536-496a-93e9-7af690fffef3`)
- **Build:** `npm run build`, publish `.next`, functions `netlify/functions`
- Um push em `master` dispara build e publicação automática (leva cerca de 1 minuto)

### O que isso exige do fluxo de trabalho

1. **Antes de começar a mexer: `git pull`.** A pasta local está atrás do
   repositório — foram feitos commits direto no GitHub que ainda não estão aí.
2. **Depois de alterar: `git add` + `git commit` + `git push origin master`.**
   É o push que publica. Sem push, a alteração fica só na máquina.
3. **Não usar mais `netlify deploy`.** Um deploy manual publica uma versão que
   não corresponde ao que está no repositório, e o próximo push sobrescreve.
4. **Rodar `npm run build` antes do push.** O build da Netlify roda
   `tsc --noEmit`; erro de tipo em qualquer arquivo do projeto reprova o deploy
   inteiro — foi o que aconteceu no primeiro deploy.

### Ao dar o primeiro `git pull`

O arquivo `scripts/tmp/jefferson.ts` será **removido da pasta local**. Ele era um
script avulso, desatualizado em relação à assinatura de `uploadDocumentToDrive`
(faltava o campo `protocol`), e reprovava o `tsc`. Nada no app o importava.
A pasta `scripts/tmp/` passou a ser ignorada pelo git, para arquivos
descartáveis não quebrarem mais o build.

---

## Commits aplicados (todos já em produção)

| Commit | O que fez |
|---|---|
| `bafcf62` | Primeiro envio do projeto ao GitHub, com `.gitignore` protegendo `.env`, `node_modules`, `.next`, `.netlify` |
| `28b10c0` | Removeu `scripts/tmp/jefferson.ts`, que quebrava o build |
| `b04a019` | Calculadora de royalties passou a somar os streams reais de cada plataforma |
| `129666a` | Calculadora legível no tema claro + colunas nomeadas + câmbio do dia preenchido |
| `5cd8e88` | Calculadora entrou no menu do portal |
| `63cafc4` | Links de rede social abrem o perfil em vez de dar 404 |

O deploy em produção hoje é o `63cafc4`.

---

## As quatro correções, em detalhe

### 1. Calculadora de royalties somava errado (`b04a019`)

A calculadora pedia **um** número de streams e o repetia nas **cinco**
plataformas antes de somar. Quem informava 100 mil streams via na tela o
dinheiro de 500 mil — a estimativa saía cinco vezes maior que a realidade.
O rodapé explicava "streams em cada plataforma", mas o número em destaque
dizia "Total estimado da sua participação", e é esse que o compositor usa
para se planejar.

Agora cada plataforma tem seu próprio campo, na ordem em que o relatório da
distribuidora entrega os números, e o total é a soma do que a pessoa
realmente tem.

Arquivo: `components/RoyaltyCalculator.tsx`

### 2. Calculadora invisível no portal do cliente (`129666a`)

O componente nasce com paleta escura. O painel admin (escopo `.p2`) já tinha
tradução para tema claro; **o portal do cliente (escopo `.pi`) não tinha
nenhuma**. Resultado: texto branco sobre fundo branco — o nome da plataforma
e o valor em reais de cada uma simplesmente não apareciam, sobrava o ícone e
a taxa por stream.

Foi adicionado um bloco `:is(.pi, .p2)` no fim de `app/hithub-ui.css` com a
paleta clara para todas as classes `royalty-calc-*`.

No mesmo commit: cabeçalho de colunas (Plataforma · Seus streams · Você
recebe), linha sem stream recuando visualmente, e a **cotação do dólar do dia
agora vem escrita no campo de câmbio** (antes era só placeholder), com um selo
dizendo "cotação de hoje" / "você alterou".

Arquivos: `components/RoyaltyCalculator.tsx`, `app/globals.css`, `app/hithub-ui.css`

### 3. Calculadora não aparecia no menu (`5cd8e88`)

A página existia e funcionava, mas nenhum menu apontava para ela — só chegava
quem passasse pelo atalho da visão geral do compositor ou pelo botão do
dashboard.

Entrou como "Simular royalties" logo depois de "Financeiro", nos dois menus,
e só para quem é compositor (a página já redireciona quem não é).

Arquivos: `app/portal/menu-do-cliente.ts`, `app/portal/compositor/menu.ts`

### 4. Link de Instagram dava 404 (`63cafc4`)

O campo pede "@ ou link", então o banco guarda de tudo: `@fabionogara`,
`fabionogara`, `instagram.com/fabionogara` e a URL inteira. Quem renderizava
jogava o valor cru no `href` — e endereço sem `https://` o navegador lê como
caminho do próprio site. O Instagram do Fábio Nogara virava
`/portal/fabionogara` e dava 404.

Foi criado `lib/links-sociais.ts` com `linkDaRede(valor, rede)`, que monta o
endereço absoluto a partir de qualquer uma dessas formas (inclusive o `@`
obrigatório na URL do TikTok). Quando não dá para montar um endereço
confiável — um `@fulano` solto no campo do Spotify, que identifica perfil por
id — devolve `null` e o botão não é desenhado.

Arquivos: `lib/links-sociais.ts` (novo), `tests/links-sociais.test.ts` (novo,
8 testes), `app/portal/artista/page.tsx`,
`app/admin/(panel)/artistas/ArtistasClient.tsx`

Suíte completa rodada depois da alteração: 344 testes, todos passando.

---

## Script novo disponível

`scripts/anexar-contrato-assinado.ts` — anexa a via assinada de um contrato e
marca o contrato como assinado pela linha de comando. Faz o mesmo que o botão
"Anexar contrato assinado" do painel (guarda o PDF, manda cópia para o Drive,
muda o status).

```
npx tsx scripts/anexar-contrato-assinado.ts --pessoa=jefferson --arquivo="caminho\do.pdf"
npx tsx scripts/anexar-contrato-assinado.ts --pessoa=jefferson --arquivo="caminho\do.pdf" --executar
```

Sem `--executar` só mostra o que faria. **O e-mail ao cliente não sai por
padrão** — só com `--avisar`. O painel avisa a pessoa porque ali tem alguém
decidindo na hora; um script roda sozinho, e mandar contrato assinado para o
e-mail de um compositor não é coisa que se faça por efeito colateral.

---

## O que ficou em aberto

### Jefferson Morais
Anexar o contrato de edição exclusiva assinado e mudar o status para
"Assinado". Pode ser pelo painel ou pelo script acima.

Observação: **não existe um campo "é exclusivo"** no cadastro. O sistema
deriva isso pela regra em `lib/repositories/portal-data.ts`:
`isExclusiveComposer` devolve true quando existe contrato
`edicao_exclusividade` com status diferente de `arquivado`. Como o contrato
dele já existe (estava "em elaboração"), ele já conta como exclusivo. O que
falta é só a via assinada e o status.

### Cinco compositores com contrato por obra indevido

Mesmo problema que foi corrigido no Rick Di Carvalho: são exclusivos, mas as
obras geraram contrato de edição por obra. O impacto é financeiro — por obra
a divisão é 75/25, na exclusividade é 80/20, então o compositor recebe menos
do que o contrato dele promete.

| Compositor | Exclusividade | Contratos por obra |
|---|---|---|
| Kaique | em elaboração | 3, todos assinados |
| Aline Fecondes | **assinada** | 2, em elaboração |
| Ariane May | em elaboração | 1, em elaboração |
| Maycon Castro | em assinatura | 1, assinado |

O script `scripts/corrigir-por-obra-de-exclusivo.ts --pessoa=<nome>` já
resolve, do mesmo jeito que resolveu o Rick.

**A Aline é o caso seguro** (exclusividade assinada, por obra ainda em
elaboração). **Kaique e Maycon são os delicados**: a exclusividade não está
assinada e o contrato por obra está — decidir qual instrumento vale é decisão
de quem assina, não de quem corrige dado.

### Suporte
Ficou uma pergunta em aberto sobre "as mensagens do suporte" — não chegou a
ser esclarecido se é um problema no código (mensagem que não chega, aviso que
não dispara) ou chamados de clientes esperando resposta.

---

## Pendência conhecida, não tratada

`npm ci` reporta 18 vulnerabilidades nas dependências (10 moderadas, 7 altas,
1 crítica). Não foram investigadas nem corrigidas — `npm audit fix --force`
traz mudanças que quebram, e isso merece uma sessão própria com os testes
rodando a cada passo.
