# LUMIO

Plataforma web de gestão financeira pessoal. Reúne controle de receitas, despesas, contas fixas, parcelamentos e empréstimos em um só lugar, com dashboards e projeção de compromissos futuros.

Projeto pessoal e independente, concebido e conduzido por mim do modelo de dados à interface.

> **Sobre este repositório:** aqui fica a documentação do projeto. O código-fonte é mantido em repositório privado.

---

## Status

O LUMIO está **no ar e em uso real**. Não é protótipo nem prova de conceito: roda em produção, com dados reais, e é o sistema que uso para controlar minhas finanças.

Segue em evolução. As melhorias entram conforme o uso revela o que falta ou o que atrapalha, e não há prazo nem escopo fechado. É um projeto independente, sem cliente, sem equipe e sem entrega contratada, o que me dá liberdade para refazer o que não ficou bom.

**Próximas melhorias**, nesta ordem:

1. **Alertas de vencimento.** Hoje o sistema projeta o compromisso futuro, mas não avisa. É a lacuna que mais incomoda no uso diário e por isso vem primeiro.
2. **Autenticação em dois fatores.** O Supabase Auth já suporta; falta habilitar e tratar o fluxo na interface.

---

## Demo

Aplicação publicada: **https://lumio.higorpaivabrito.workers.dev/**

O acesso exige login. Use a conta de demonstração abaixo:

```
Usuário: teste@lumio.dev
Senha:   Teste@2026
```

A conta de demonstração contém dados fictícios e é isolada das demais por Row Level Security.

---

## Telas

<!-- PRINT: Dashboard completo -->
**Dashboard** - saldo projetado, entradas e saídas do período, orçamento comprometido, indicador de saúde financeira e gráfico de fluxo mensal.

<!-- PRINT: Movimentações -->
**Movimentações** - lançamentos do período selecionado, com confirmação e edição individual.

<!-- PRINT: Cadastro -->
**Cadastro** - contas fixas, à vista e parceladas, beneficiários e categorias.

<!-- PRINT: Simulador -->
**Simulador** - projeção do impacto de um novo compromisso no orçamento.

<!-- PRINT: Configurações -->
**Configurações** - personalização de tema e preferências do usuário.

---

## O problema

Planilhas de controle financeiro resolvem o registro, mas falham em três pontos: não projetam o futuro, não avisam nada e ficam ilegíveis quando entram parcelamentos e contas recorrentes.

O LUMIO foi construído em cima dessas três lacunas. A ideia central é unir controle, visualização e planejamento no mesmo fluxo: o que você lança hoje já aparece na projeção dos próximos meses.

---

## Funcionalidades

**Contas e lançamentos**
- Três tipos de conta: **Fixa** (recorrente), **À vista** (pontual) e **Parcelada** (com controle de parcelas)
- Contas fixas geram ocorrências mensais futuras automaticamente; apenas o mês corrente carrega status real, os meses seguintes ficam marcados como projeção
- Edição por ocorrência: alterar um mês específico de uma conta fixa não altera o histórico nem os demais meses
- Beneficiários e categorias personalizáveis, com ícones próprios

**Visualização**
- Dashboard com saldo projetado, entradas, saídas e orçamento comprometido
- Indicador de saúde financeira com explicação dos números que o compõem
- Gráficos de fluxo mensal e histórico, desenhados em SVG
- Filtro de período em todas as telas: mês atual, próximo mês, 3/6/12/24 meses ou intervalo personalizado

**Planejamento**
- Simulador de impacto de novos compromissos
- Projeção de compromissos futuros a partir das contas cadastradas

**Gestão**
- Temas e paletas de cores selecionáveis
- Tela de bloqueio por inatividade, com tempo configurável

---

## Arquitetura

```
Navegador (HTML/CSS/JavaScript)
        |
        v
Supabase Auth  ->  sessão do usuário
        |
        v
Supabase (PostgreSQL)  ->  Row Level Security por usuário
        |
        v
Edge Functions  ->  regras de negócio mais pesadas
```

Publicação em **Cloudflare Workers**.

### Stack

| Camada | Tecnologia |
|---|---|
| Frontend | JavaScript, HTML e CSS puros, sem frameworks |
| Gráficos | SVG escrito à mão, sem bibliotecas de charting |
| Autenticação | Supabase Auth (e-mail e senha) |
| Banco de dados | PostgreSQL via Supabase |
| Regras de negócio | Supabase Edge Functions |
| Publicação | Cloudflare Workers |
| Versionamento | Git / GitHub |

### Modelo de dados

Esquema relacional com isolamento por usuário aplicado no banco, não na aplicação:

| Tabela | Conteúdo |
|---|---|
| `beneficiarios` | pessoas ou entidades vinculadas aos lançamentos |
| `categorias` | classificação de receitas e despesas, com ícone |
| `contas` | contas fixas, à vista e parceladas |
| `parcelas` | parcelas geradas a partir de contas parceladas |
| `excecoes_fixa` | alterações pontuais em um mês específico de uma conta fixa |
| `avulsos` | lançamentos únicos, fora do cadastro de contas |
| `configuracoes` | preferências do usuário |

Toda leitura e escrita passa por políticas de **Row Level Security**: cada usuário só enxerga as próprias linhas, independentemente do que a aplicação peça.

---

## Como o projeto foi construído

O LUMIO nasceu de um problema meu e foi conduzido por mim do começo ao fim. O produto, o modelo de dados, as regras de negócio e as decisões de arquitetura são minhas. A escrita do código foi feita em conjunto com o **Claude (Anthropic), modelo Opus**, com a IA no papel de implementadora e eu no papel de quem define o que o sistema faz, valida o resultado e decide o que entra.

Registro isso porque acho que faz diferença saber como uma coisa foi feita. E porque o trabalho que sobra depois de tirar a digitação do código é justamente o que me interessa: entender o problema, modelar, escolher entre soluções, testar em uso real, encontrar o que quebrou e decidir como corrigir. Boa parte das decisões documentadas na seção seguinte veio exatamente daí, de erro encontrado em produção.

---

## Decisões técnicas

**Zero dependências no frontend.**
Sem framework, sem biblioteca de CSS, sem biblioteca de gráficos. Os charts são SVG gerado a partir dos dados, e o design system é feito com tokens CSS próprios. A motivação foi entender de fato o que acontece em cada camada, em vez de herdar comportamento pronto, e o resultado é um bundle sem árvore de dependências para manter.

**Migração de Google Apps Script para Supabase.**
A primeira versão rodava sobre Google Apps Script com o Google Sheets fazendo papel de banco. Funcionou até o ponto em que as limitações apareceram: latência alta em operações de escrita, ausência de integridade referencial e um modelo de permissão que não isolava usuários de forma confiável. A migração para PostgreSQL trouxe chaves estrangeiras, RLS e tempos de resposta compatíveis com uso real.

**Exceções em contas recorrentes.**
Contas fixas projetam meses à frente, mas a vida altera um mês isolado: a conta de luz que veio mais cara, o mês em que a mensalidade não foi paga. Em vez de duplicar registros ou reescrever a série inteira, cada alteração pontual vira uma linha em `excecoes_fixa`, e a regeneração da série preserva os meses já marcados como exceção. O histórico passado nunca é reescrito por uma edição feita hoje.

**Janelas de período com limite inferior.**
Os filtros de período foram reescritos depois que os totais do dashboard passaram a somar tudo desde o início da base. A causa era um filtro com data final, mas sem data inicial. Hoje toda opção de horizonte é uma janela fechada, com início e fim explícitos, e essa regra é a mesma para dashboard, movimentações e cadastro.

**Interface própria em vez de componentes nativos.**
Diálogos de confirmação e alerta do navegador foram substituídos por modais próprios, seguindo o tema ativo. Além da consistência visual, isso permite mensagens específicas do domínio em vez de texto genérico.

---

## Contato

**Higor Paiva**
[LinkedIn](https://www.linkedin.com/in/higor-paiva) · [GitHub](https://github.com/paivahp) · higorpaivabrito@gmail.com
