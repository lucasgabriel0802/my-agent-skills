---
name: commit-detalhado
description: "Use sempre que o usuário pedir para gerar, revisar ou melhorar uma mensagem de commit git. Acionar para frases como 'gera o commit', 'mensagem de commit', 'escreve o commit', 'como commitar essa mudança', ou quando o usuário descrever o que mudou no código e precisar registrar. Inclui verificação obrigatória de segurança e dados sensíveis antes de commitar. Produz commits em Português seguindo Conventional Commits, com título, corpo explicativo, arquivos alterados e impacto/breaking changes."
---

# Commit Detalhado — Padrão de Mensagem Git

Esta skill define como gerar mensagens de commit claras, completas e padronizadas em Português, seguindo a convenção Conventional Commits.

---

## Estrutura obrigatória de todo commit

```
<tipo>(<escopo opcional>): <título>

<corpo — o porquê da mudança>

Arquivos alterados:
- <arquivo ou módulo>: <o que mudou nele>

Impacto:
<consequências da mudança; se houver breaking change, indicar explicitamente>
```

---

## Tipos (Conventional Commits)

| Tipo | Quando usar |
|------|-------------|
| `feat` | Nova funcionalidade para o usuário |
| `fix` | Correção de bug |
| `refactor` | Refatoração sem mudança de comportamento |
| `chore` | Tarefas de manutenção, configs, dependências |
| `docs` | Alterações apenas em documentação |
| `style` | Formatação, espaçamento, sem lógica alterada |
| `test` | Adição ou correção de testes |
| `perf` | Melhoria de performance |
| `ci` | Mudanças em pipeline/CI |
| `revert` | Reversão de commit anterior |

---

## Regras do título

- Máximo **72 caracteres**
- Começar com **letra minúscula** após os dois-pontos (ex: `fix(login): corrigir validação de senha`)
- Usar **imperativo presente** ("corrigir", "adicionar", "remover" — não "corrigido" ou "adicionando")
- Escopo entre parênteses é opcional, mas recomendado quando a mudança é localizada (ex: `feat(relatorio)`, `fix(datamodule)`)
- Não terminar com ponto

## Regras do corpo

- Separado do título por **uma linha em branco**
- Explicar **o porquê** da mudança, não apenas o quê (o quê já está no título)
- Contexto útil: qual problema existia, qual a causa raiz, por que essa abordagem foi escolhida
- Sem limite rígido de linhas, mas objetivo e direto
- Cada linha do corpo: máximo **100 caracteres**

## Regras de arquivos alterados

- Listar os arquivos/units/módulos principais afetados
- Uma linha por arquivo no formato `- NomeDoArquivo: descrição da mudança`
- Não listar arquivos triviais (ex: apenas formatação automática), focar no que tem relevância

## Regras de impacto

- Descrever efeitos colaterais relevantes (ex: mudança de comportamento em outra tela, necessidade de rodar migration, necessidade de recompilar projeto)
- Se houver **breaking change** (quebra de compatibilidade), destacar explicitamente com `BREAKING CHANGE:` em caixa alta
- Se não houver impacto relevante além da mudança descrita, escrever `Sem impacto em outras áreas.`

---

## 🔒 Verificação Obrigatória de Segurança e Dados Sensíveis

Antes de gerar qualquer mensagem ou efetivar o commit, é **obrigatório inspecionar o diff** para garantir que nenhum dado confidencial esteja sendo adicionado acidentalmente:

- **Credenciais e Senhas**: senhas *hardcoded*, senhas de bancos de dados (ex: `masterkey`, etc.), logins e senhas de teste.
- **Segredos e Tokens**: chaves de API (*API keys*), *secrets* de serviços de terceiros, tokens JWT, *bearer tokens*, chaves de webhook.
- **Certificados e Chaves Privadas**: arquivos `.pfx`, `.p12`, `.key`, `.pem`, ou senhas de certificados digitais A1/A3.
- **Dados Pessoais ou de Clientes (LGPD)**: CPFs, CNPJs reais em arquivos de teste/seeders, dados bancários, números de cartão, nomes e contatos de clientes.
- **Infraestrutura e Redes**: *connection strings* completas de produção/homologação, IPs internos, URLs privadas com credenciais embutidas.
- **Arquivos Temporários ou de Ambiente**: arquivos `.env`, dumps de banco de dados (`.sql`, `.fdb`), logs contendo payloads reais ou arquivos `.ini` com parâmetros sigilosos.

> [!CAUTION]
> **Se qualquer dado sensível for detectado no diff, o commit deve ser IMEDIATAMENTE INTERROMPIDO**, alertando o usuário sobre o arquivo e a linha afetada para que o conteúdo seja removido antes de prosseguir.

---

## Processo ao gerar um commit

- **Manter commits granulares e pequenos**: Um commit por responsabilidade (ex: um commit para lógica de negócio, outro para UI).
- **Sempre confirmar com o usuário**: Sugerir as mensagens de commit e aguardar a aprovação do usuário para efetivar.

1. **Verificação de segurança** — inspecionar o diff para garantir que não há dados sensíveis, credenciais ou segredos antes de prosseguir
2. **Entender a mudança** — ler o diff ou a descrição fornecida pelo usuário antes de escrever qualquer coisa
3. **Classificar o tipo** — escolher o tipo Conventional Commits que melhor representa a intenção
4. **Redigir o título** — claro, no imperativo, dentro de 72 caracteres
5. **Escrever o corpo** — focar no porquê, com contexto suficiente para que outra pessoa entenda meses depois
6. **Listar arquivos** — apenas os relevantes
7. **Avaliar impacto** — pensar se a mudança afeta algo além do ponto alterado

Se a descrição do usuário for vaga (ex: "fiz umas correções"), **perguntar antes de gerar** o que exatamente foi alterado e qual era o problema — um commit vago não serve de nada no histórico.

---

## Exemplos

### Exemplo 1 — Correção de bug
```
fix(nfe): corrigir cálculo de ICMS para operações interestaduais

O percentual de redução de base de cálculo não estava sendo aplicado
em notas com CFOP de venda interestadual, resultando em valor de ICMS
incorreto. A lógica foi ajustada para verificar o tipo de operação
antes de aplicar a alíquota.

Arquivos alterados:
- dmNFe.pas: corrigida função CalculaICMS para considerar CFOP interestadual
- uNFe.pas: ajustada chamada de CalculaICMS passando o CFOP corretamente

Impacto:
Notas fiscais interestaduais emitidas antes desta correção podem ter
valores incorretos. Não há breaking change na interface da unit.
```

### Exemplo 2 — Nova funcionalidade
```
feat(relatorio): adicionar filtro por período no relatório de vendas

O relatório de vendas não permitia filtrar por data, exibindo sempre
todos os registros. Adicionados campos de data inicial e final com
validação para garantir que o período seja consistente antes de
executar a consulta.

Arquivos alterados:
- frmRelatorioVendas.pas: adicionados controles de filtro de data e validação
- dmRelatorioVendas.pas: query de vendas atualizada com parâmetros de período

Impacto:
Sem breaking change. O relatório sem filtro continua funcionando
com os campos de data em branco (comportamento padrão mantido).
```

### Exemplo 3 — Refatoração
```
refactor(cliente): extrair validações de CPF/CNPJ para unit separada

As validações de CPF e CNPJ estavam duplicadas em três formulários
diferentes (frmCliente, frmFornecedor, frmTransportadora), causando
inconsistências ao longo do tempo. A lógica foi centralizada na
unit uValidacoes para facilitar manutenção e garantir consistência.

Arquivos alterados:
- uValidacoes.pas: criada com funções ValidarCPF e ValidarCNPJ
- frmCliente.pas: substituída lógica local pela chamada a uValidacoes
- frmFornecedor.pas: idem
- frmTransportadora.pas: idem

Impacto:
Sem breaking change. Comportamento de validação mantido idêntico.
Adicionar uValidacoes ao uses de qualquer unit que precisar validar
documentos no futuro.
```
