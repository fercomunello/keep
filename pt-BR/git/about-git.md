# Conceitos de Git

## Índice 
- [O que é Git?](#o-que-é-git)
    * [Quem inventou o Git?](#quem-inventou-o-git)
    * [Comandos básicos](#comandos-mais-básicos-do-git)
    * [Diferencial do Git](#qual-é-o-diferencial-do-git)
- [O Git se tornou um padrão](#o-git-se-tornou-um-padrão)
    * [Desempenho e flexibilidade](#desempenho-e-flexibilidade)
- [Perguntas frequentes](#faq)
    * [Git é complicado?](#git-é-complicado)
    * [Devo aprender Git?](#uso-outro-sistema-de-vcs-devo-aprender-git)
    * [Como migrar o sistema de VCS?](#como-migrar-de-outro-sistema-de-vcs-para-git)
    * [Onde aprender Git?](#onde-posso-aprender-git)
    * [Esqueci um comando e agora?](#tenho-pressa-ou-esqueci-algum-comando-e-agora)
    * [Terminal vs GUI](#preciso-sempre-executar-os-comandos--na-mão--via-cli)

## O que é Git?
Tendo uma arquitetura distribuída, o Git é um exemplo de DVCS (portanto, Sistema de Controle de Versão Distribuído). Em vez de ter apenas um único local para o histórico completo da versão do software, como é comum em sistemas de controle de versão outrora populares como CVS ou Subversion (também conhecido como SVN), no Git, a cópia de trabalho de todo desenvolvedor do código também é um repositório que pode conter o histórico completo de todas as alterações.

### Quem inventou o Git?
O Git foi idealizado originalmente por [Linus Torvalds](https://pt.wikipedia.org/wiki/Linus_Torvalds) em 2005 para 
fins de desenvolvimento do [Kernel Linux](https://github.com/torvalds/linux), juntamente com a contribuição 
de outros desenvolvedores do kernel.

Desde então, o Git é mantido por muitos desenvolvedores de todas as partes do mundo.
> Inclusive, o código do Git é mantindo neste repositório aberto 
> com dezenas de milhares de commits: https://github.com/git/git

### Comandos mais básicos do Git
Consulte o [cheat sheet (folha de dicas)](https://training.github.com/downloads/pt_BR/github-git-cheat-sheet.pdf) 
oficial do Git.

### Qual é o diferencial do Git?
Além de ser distribuído, o Git foi projetado com desempenho, segurança e flexibilidade em mente.

## O Git se tornou um padrão
Um grande número de desenvolvedores já tem experiência com o Git e uma proporção significativa de recém-formados pode 
ter experiência apenas com o Git. Embora algumas empresas precisem escalar a curva de aprendizado ao migrar para o Git 
de outro sistema de versionamento de código, muitos desenvolvedores existentes e futuros não precisam ser treinados no Git.

### Desempenho e flexibilidade
As características brutas de desempenho do Git são muito fortes quando comparadas a muitas alternativas. Fazer o commit
de novas alterações, branches, mesclagem e comparação de versões anteriores – tudo é otimizado para desempenho.
Os algoritmos implementados no Git aproveitam o conhecimento profundo sobre atributos comuns de árvores de arquivos de
código-fonte reais, como costumam ser modificados ao longo do tempo e quais são os padrões de acesso.

Além dos benefícios de um grande conjunto de talentos, a predominância do Git também significa que muitas ferramentas e 
serviços de software de terceiros já estão integrados ao Git.

Se você é um desenvolvedor inexperiente que quer desenvolver habilidades valiosas em ferramentas de desenvolvimento de
software, quando se trata de controle de versão, o Git deve ser um dos itens na lista.


## FAQ

### Git é complicado?
Uma crítica comum ao Git é que pode ser difícil de aprender. Algumas das terminologias do Git vão ser novas para os
iniciantes e, para usuários de outros sistemas, a terminologia do Git pode ser diferente, por exemplo, revert no Git 
tem um significado diferente do que no SVN ou CVS. No entanto, o Git é muito capaz e disponibiliza muitos recursos 
robustos aos usuários.

### Uso outro sistema de VCS, devo aprender Git?
Para as equipes que vêm de um VCS não distribuído, ter um repositório central pode parecer uma coisa boa que eles não 
querem perder. No entanto, embora o Git tenha sido projetado como um sistema de controle de versão distribuído (DVCS), 
com o Git, você ainda pode ter um repositório canônico oficial em que todas as alterações no software devem ser armazenadas.

Com o Git, como o repositório de cada desenvolvedor está completo, o trabalho não precisa ser restringido pela 
disponibilidade e desempenho do servidor "central". Durante interrupções ou quando offline, os desenvolvedores 
ainda podem consultar o histórico completo do projeto.

### Como migrar de outro sistema de VCS para Git?
Se você usa o Subversion como sistema de versionamento na sua empresa, consulte o
[guia de migração oficial](https://training.github.com/downloads/subversion-migration/) da ferramenta.

### Onde posso aprender Git?
Sem dúvidas, o melhor lugar para aprender é através da [documentação oficial](https://git-scm.com/docs).

### Tenho pressa ou esqueci algum comando, e agora?
Sem problemas! Neste repositório, disponibilizei um guia mais resumido com os comandos principais do Git,
aqueles comandos que são mais usados no dia-a-dia.

[Guia: Principais comandos do Git](git-commands.md)

Se você ainda tem dúvidas sobre algum conceito ou gosta de conteúdo em vídeo, recomendo assistir 
essa [playlist de vídeo-aulas](https://www.youtube.com/playlist?list=PLuYctAHjg89bR5PgaAlyGCl2PWMPDMzFN) 
do [Rinaldo (Red Hat)](https://github.com/rinaldodev) que foram publicadas no YouTube.

### Preciso sempre executar os comandos "na mão" via CLI?
Não necessariamente, você pode utilizar [ferramentas gráficas](https://git-scm.com/downloads/guis) (GUI) que auxiliam 
a realizar desde operações básicas a operações avançadas num repositório Git, ou até mesmo as ferramentas integradas 
na sua própria IDE de programação como é o caso das [IDEs da JetBrains](https://www.jetbrains.com/pt-br/products/). 

No entanto, vale ressaltar que saber como o Git funciona por trás dos panos é mais importante do que decorar
os comandos ou tentar automatizar as operações usando uma GUI, por isso alguns desenvolvedores preferem
executar os comandos no terminal explicitamente.

### Outras perguntas frequentes
Consulte o FAQ oficial: https://git-scm.com/docs/gitfaq

