# Contribuições

---

## Por onde começar?

1. Faça o _fork_ do projeto;
2. Entenda o fluxo de contribuições;
3. Leia e pratique as boas práticas;

---

## Fluxo

É muito fácil contribuir para o projeto e iniciantes no universo _Open Source_ são mais do que bem-vindas(os)!

Qualquer tipo de ajuda (seja ela grande ou pequena) é válida e construtiva. Se encontrar qualquer parte do livro que possa ser melhorada, essa é uma grande oportunidade para participar. 

E caso não saiba por onde começar:

1. Faça referência ao repositório oficial após o fork

`git remote add upstream git@github.com:okrdu/sicp-ptbr.git`

2. Antes de iniciar o processo de contribuição, crie uma nova branch para fazer suas alterações.

Alguns exemplos:

- Para tradução: `git checkout -b traducaoCapX`
- Para revisões: `git checkout -b revisaoCapX`
- Para erros: `git checkout -b correcaoCapX`

Use qualquer nome que seja coerente com a contribuição que está sendo feita. X representa o número do capítulo.

3. Após realizar as alterações, é hora de fazer um commit com uma mensagem coerente do que foi feito. Exemplo:

`git add --all` <br>
`git commit -am ‘Adiciona tradução/revisão/melhoria capítulo X linha/linhas Y` <br>
`git push origin traducaoCapX` <br>

4. Envie um Pull Request com as alterações feitas, fazendo referência para o master do repositório oficial.

5. Sua contribuição será analisada pela comunidade. Em alguns casos pediremos algumas alterações antes de dar merge.

Após o merge:

Delete a branch utilizada:

`git checkout master` <br>
`git push origin traducaoCapX` <br>
`git branch -D traducaoCapX`

Atualize seu repositório com o repositório oficial:

`git fetch upstream` <br>
`git rebase upstream/master` <br>
`git push -f origin master`

6. Quando iniciar uma nova contribuição, inicie repita o processo pelo passo 2.

---

## Boas Práticas

- Não traduza termos técnicos e blocos de código
- Antes de enviar sua contribuição, certifique-se de que está enviando apenas um único _commit_ que represente o que foi feito. Caso tenha feito vários commits, [esmague-os](http://gitready.com/advanced/2009/02/10/squashing-commits-with-rebase.html) antes de fazer o _Pull Request_.
- Caso tenha qualquer tipo de dúvida, abra uma _Issue_ que faremos o possível para te ajudar.
- Contribua com as discussões.
- Estrangeirismo: Utilizar o formato itálico. Exemplo: _bug_
- Sentido Figurado: Sempre destacar com aspas duplas.
- Citação: Aspas duplas com o sinal de >. Exemplo:
> “Foo bar”
- Marcação para código: Utilizar um apóstrofe (`) para indicar um pedaço de código no meio de um texto (var foo = undefined). Ou três apóstrofe com o nome da linguagem de programação na frente (```scheme), para indicar um bloco de código:

```scheme
(display "Hello, World!")
```

---

_Esse texto é um documento em constante edição, e pode ser alterado no futuro._

Um agradecimento especial ao [BrazilJS](https://github.com/braziljs/eloquente-javascript) por possibilitar a adaptação de sua documentação!

_[<--- Voltar à Página Inicial](https://github.com/okrdu/sicp-ptbr)_