# Como publicar o win95 no VS Code Marketplace

Guia manual para lançar uma nova versão da extensão `asilva.win95`.

- Publisher: `asilva` (<https://marketplace.visualstudio.com/manage/publishers/asilva>)
- Página da extensão: <https://marketplace.visualstudio.com/items?itemName=asilva.win95>
- Ferramenta: [`@vscode/vsce`](https://github.com/microsoft/vscode-vsce), rodada com `npx` (não precisa instalar)

> **Não use `npm run deploy`.** O script do `package.json` usa o pacote antigo `vsce` e depende do yarn, que não está instalado.

Existem dois jeitos de publicar:

| | Opção A: `vsce publish` | Opção B: upload do `.vsix` no site |
|---|---|---|
| Precisa de PAT | Sim | Não |
| Funciona depois de 1/12/2026 | **Não** (PATs globais são desativados nessa data) | Sim |

---

## 1. Preparar a versão

1. Aumente a versão no `package.json`. O Marketplace recusa uma versão que já foi publicada.
   ```sh
   npm version patch --no-git-tag-version   # 2.0.0 -> 2.0.1 (use minor ou major se fizer sentido)
   ```
2. Adicione uma entrada no `CHANGELOG.md`. Ela aparece na aba Changelog da loja.
3. Se mudou o visual, atualize a print (veja [Atualizar a print](#atualizar-a-print)).
4. Teste o pacote localmente:
   ```sh
   npx @vscode/vsce package
   code --install-extension win95-*.vsix   # opcional: instala e confere o tema
   ```
   O comando `package` lista os arquivos incluídos. O pacote deve ter poucos KB; as imagens `example*.png` ficam de fora pelo `.vscodeignore`.
5. **Faça commit e push para a `master` antes de publicar.** Na loja, as imagens do README são carregadas de `github.com/arxdsilva/win95/raw/HEAD/...`. Sem o push, a imagem aparece quebrada.
   ```sh
   git add package.json CHANGELOG.md README.md example-v*.png
   git commit -m "release v2.0.1"
   git tag v2.0.1
   git push && git push origin v2.0.1
   ```

---

## 2A. Publicar com PAT (até 30/11/2026)

### Criar o PAT

1. Acesse <https://dev.azure.com> com a **mesma conta Microsoft dona do publisher `asilva`**.
   - Se pedir para criar uma organização, crie qualquer uma. O token só é gerado dentro de uma organização.
   - Se a página ficar voltando para o login, use uma janela anônima e entre só com essa conta.
2. No canto superior direito, clique no ícone de usuário e depois em **User settings → Personal access tokens**.
3. Clique em **+ New Token** e preencha:
   - **Name:** `vsce-win95`
   - **Organization:** **All accessible organizations**. Isso é obrigatório: com uma organização específica, o `vsce` retorna `401 Unauthorized`.
   - **Expiration:** 30 dias (o suficiente para o lançamento)
   - **Scopes:** **Custom defined** → **Show all scopes** → **Marketplace** → marque **Manage**
4. Clique em **Create** e **copie o token na hora**. Ele não aparece de novo.
   Não salve o token em arquivo dentro do repositório.

### Publicar

```sh
npx @vscode/vsce login asilva    # cole o PAT quando pedir
npx @vscode/vsce publish         # publica a versão do package.json
```

Quando terminar, se quiser remover o token salvo na máquina:

```sh
npx @vscode/vsce logout asilva
```

Depois revogue o token em **User settings → Personal access tokens → Revoke**.

---

## 2B. Publicar pelo site (sem PAT, funciona sempre)

1. Gere o pacote:
   ```sh
   npx @vscode/vsce package      # cria win95-<versão>.vsix
   ```
2. Acesse <https://marketplace.visualstudio.com/manage/publishers/asilva> com a conta dona do publisher.
3. Na linha da extensão **win95**, clique em **⋯ → Update**.
4. Envie o arquivo `win95-<versão>.vsix` e confirme.

---

## 3. Conferir

- A loja faz uma verificação antes de liberar, o que pode levar alguns minutos.
- Confira a versão publicada:
  ```sh
  npx @vscode/vsce show asilva.win95
  ```
- Abra a [página da extensão](https://marketplace.visualstudio.com/items?itemName=asilva.win95) e confira a print, a descrição e o changelog.
- No VS Code, a atualização chega para os usuários em algumas horas.

---

## Atualizar a print

1. Abra no VS Code um projeto de exemplo, sem nada pessoal (nomes de clientes, caminhos, histórico do terminal), com o tema `win95` ativo.
2. Tire a print da janela com `Cmd+Shift+4`, aperte `Espaço` e clique na janela.
3. Salve com **um nome novo** (`example-v4.png`, `example-v5.png`…). Com o mesmo nome, o GitHub e a loja podem continuar mostrando a imagem antiga do cache.
4. Troque o nome da imagem no `README.md`.
5. Não use SVG: o Marketplace recusa imagens SVG no README.

---

## Problemas comuns

| Erro | Causa / solução |
|---|---|
| `401 Unauthorized` / `Access Denied` | PAT sem **All accessible organizations**, sem o escopo **Marketplace → Manage**, criado com outra conta, ou já expirado. Crie outro. |
| `... already exists` | A versão do `package.json` já foi publicada. Aumente a versão. |
| Print quebrada na loja | O commit com a imagem não foi enviado para a `master`. Faça o push. |
| `Make sure to edit the README.md file` | O README ainda tem o texto do template. Edite. |
| PAT parou de funcionar depois de 1/12/2026 | PATs globais foram desativados. Use a [Opção B](#2b-publicar-pelo-site-sem-pat-funciona-sempre). |
