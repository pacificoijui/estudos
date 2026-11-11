# Estudos do Pedrinho

App de estudos em um arquivo só (`index.html`): matérias, submatérias com anotações,
questões e rodadas de prática.

## Como os dados são guardados

1. **Sempre no aparelho.** Tudo cai no `localStorage` na hora, então o app funciona
   offline e nunca depende da internet para salvar.
2. **Na nuvem, quando você entra.** Com login, os dados também vão para o Firestore
   e voltam para os outros aparelhos em tempo real — o que você escreve no celular
   aparece no computador sem recarregar a página.

Celular e computador precisam usar **a mesma conta** (mesmo e-mail e senha).

### Juntar em vez de sobrescrever

Os dois lados são mesclados por `id` e data de alteração:

- cada registro guarda `atualizadoEm`; na dúvida, vence a versão mais nova;
- exclusões deixam uma marca em `apagados` (com a hora), então o que você apagou
  num aparelho não volta pelo outro;
- escrever offline nos dois aparelhos não perde nada: quando os dois se conectam,
  as duas listas se juntam.

O botão **Importar** também junta com o que já existe (não apaga o resto), e o que
vem do backup vale mais que exclusões anteriores.

### Se a nuvem não carregar

O Firebase fica num `<script type="module">` separado de propósito. Sem internet,
com a rede bloqueada ou se o CDN cair, o app continua funcionando normal, só que
salvando apenas naquele aparelho — o rodapé (⚙ no celular) mostra em que estado está.

## Configuração do Firebase

O projeto usado é `estudos-concurso-9ed60`. A `apiKey` que aparece no `index.html`
é pública por natureza (todo app web do Firebase expõe a dela) — **quem protege os
dados são as regras do Firestore**, não a chave.

O app grava em `usuarios/{uid}`, um documento por conta. Se as regras não
permitirem esse caminho, ele volta sozinho para `estudos/dados`, que é onde os
dados moravam antes (a migração é automática na primeira vez).

Regras recomendadas (Console do Firebase → Firestore → Regras):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // cada conta só enxerga o próprio documento
    match /usuarios/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
    // documento antigo, mantido só para a migração
    match /estudos/dados {
      allow read, write: if request.auth != null;
    }
  }
}
```

Como o cadastro por e-mail/senha fica aberto por padrão, qualquer pessoa com o link
pode criar uma conta. Com as regras acima isso não dá acesso aos seus dados — cada
conta nova começa vazia. Para fechar de vez, é só desativar o provedor
"E-mail/senha" no Console (Authentication → Sign-in method) depois de criar as suas
contas, ou apagar o `match /estudos/dados` quando a migração já tiver acontecido.

## Backup em arquivo

O botão **Baixar backup** (rodapé no computador, ⚙ no celular) gera um `.json`
com tudo: matérias, submatérias, anotações, questões e histórico. É a rede de
segurança que vale mesmo que a conta, o Firebase ou o navegador sumam.

O arquivo começa com um bloco `_backup` — data, totais e como restaurar — e o
resto são os dados em si. **Restaurar de um arquivo** lê de volta e *junta* com o
que já existe, em vez de apagar; o que vem do backup vale mais que exclusões
anteriores, então serve para recuperar algo apagado por engano.

O app guarda a data do último backup e, passados 14 dias (ou se nunca houve um),
mostra um aviso na tela de Matérias com o botão para baixar na hora.

## Largura da coluna (computador)

No computador a coluna de conteúdo tem largura própria (680px) e encosta à
esquerda, em vez de esticar até o fim da tela — assim sobra espaço à direita para
deixar o vídeo da aula em popup por cima do desktop, sem cobrir nada.

Para ajustar, arraste a borda direita da caixa; dois cliques nela voltam ao
padrão. A escolha fica guardada **só naquele aparelho** (depende do tamanho da
tela, então não faz sentido sincronizar). No celular nada disso se aplica: a
caixa continua ocupando a largura toda.

## Anotações: ler por padrão, editar quando quiser

A submatéria abre em **modo leitura** — texto formatado, sem editor e sem nada
clicável por acidente. O botão **✎ Editar** abre a barra de formatação e o campo
de texto; **Pronto** salva e volta para a leitura (durante a edição também salva
sozinho a cada tecla).

As anotações guardam HTML (negrito, itálico, sublinhado, marca-texto, título,
listas e citação). A barra de formatação fica fixa no topo do editor e funciona no
toque — o botão segura a seleção para o teclado do celular não fechar.

## Excluir

Os botões de excluir matéria e submatéria não ficam soltos na tela, porque era
fácil encostar neles sem querer no celular. Eles vivem dentro do modal de
**✎ Editar** (no canto oposto ao Salvar) e ainda pedem confirmação.

Anotações antigas, em texto puro, continuam abrindo normalmente: elas passam a ser
HTML na primeira vez que você editar (`fmt: 'html'` marca as convertidas).

Todo HTML — o que vem da nuvem e o que você cola — passa por uma peneira que só
deixa passar marcação de formatação. Script, imagem, `onerror` e link
`javascript:` são removidos antes de aparecer na tela.
