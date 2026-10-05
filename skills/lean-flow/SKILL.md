---
name: lean-flow
description: "Fluxo de desenvolvimento e correção cirúrgica de alta precisão e baixo consumo de tokens. Inspirado no método de Matt Pocock com o pragmatismo Ponytail. Aplica sabatinada seletiva (máx 2-3 perguntas objetivas), Micro-Spec (<25 linhas) com raio de ação (Blast Radius), TDD com verificação factual em terminal e correção de bugs/erros de tela na causa raiz pelo menor diff funcional."
---

# Lean Flow — Precisão Cirúrgica & Mínimo Consumo de Tokens

Este método combina a disciplina orientada a especificações e testes (Matt Pocock) com o pragmatismo de menor diff funcional e correção na causa raiz (Ponytail), otimizado especificamente para **evitar o desperdício de tokens** e maximizar a assertividade do modelo de IA.

---

## Princípio Fundamental: Fato no Terminal vs. Alucinação no Chat

- **Output prolixo é caro e lento:** Nunca gere redações, explicações didáticas óbvias ou PRDs corporativos longos.
- **Validação factual no terminal vence qualquer raciocínio simulado:** Rodar um comando local (`pest`, `phpunit`, `artisan test`, etc.) gasta centavos de tokens e devolve a verdade factual; tentar "adivinhar" o comportamento de 10 arquivos gasta milhares de tokens de raciocínio e frequentemente falha.
- **Leitura JIT (Just-in-Time):** Nunca leia arquivos inteiros sem necessidade. Use ferramentas de busca cirúrgica (`ripgrep`, recortes com `StartLine`/`EndLine`).

---

## Roteamento de Modo

Identifique imediatamente o cenário antes de agir:

```
[Entrada do Usuário]
  ├── Nova funcionalidade ou Refatoração não-trivial  ──> MODO 1: FEATURE / REFACTOR
  ├── Erro, Bug ou Stack Trace colada da tela        ──> MODO 2: FIX / SCREEN-ERROR
  └── Ajuste simples / óbvio (1-5 linhas, sem dúvida) ──> MODO 3: TRIVIAL (Execução Direta)
```

---

## MODO 1: FEATURE / REFACTOR (Planejado & Econômico)

### 1. Targeted Grill (Sabatinada Seletiva)
- **Regra:** Máximo de 2 a 3 perguntas de múltipla escolha ou fechadas (sim/não).
- **Foco exclusivo:** Pontos cegos de arquitetura, permissões, limites de negócio ou efeitos colaterais em dados existentes.
- **Proibido:** Perguntas abertas genéricas ou filosóficas.
- Se o escopo já estiver 100% claro e sem ambiguidades, **pule diretamente para a Micro-Spec**.

### 2. Micro-Spec (< 25 linhas)
Apresente uma especificação enxuta e aguarde confirmação rápida do usuário antes de codificar:

```markdown
### Micro-Spec
- **Alvo:** [1 linha explicando o que será entregue]
- **Blast Radius (Raio de Ação):**
  - *Arquivos a alterar:* `app/Policies/TransferPolicy.php`, `...`
  - *Arquivos estritamente protegidos:* [o que NÃO deve ser tocado]
- **Critério de Aceitação / Comando:** `php artisan test --filter=TransferTest`
```

### 3. Tracer Bullet & TDD
1. Crie ou atualize o teste automatizado mínimo que expressa a nova regra.
2. Execute o teste no terminal (Red).
3. Implemente o código mínimo estritamente necessário (Green).
4. Reexecute o comando no terminal. Passou? Conclua a tarefa sem firulas.

---

## MODO 2: FIX / SCREEN-ERROR (Diagnóstico Cirúrgico de Causa Raiz)

Use sempre que o usuário relatar um bug, comportamento inesperado ou colar um erro/stack trace de tela.

### 1. Filtro da Stack Trace & Localização
- Ignore completamente frames de framework/bibliotecas (`vendor/`, `node_modules/`).
- Localize o **primeiro frame do código da aplicação**.
- Leia apenas as 15 a 25 linhas ao redor da falha usando ranges precisos.

### 2. Diagnóstico da Discrepância ("Por que o teste passou se a tela quebrou?")
Se os testes automatizados já existiam e estavam passando enquanto a tela explodiu, responda em 1 linha:
- Qual dado real enviado pelo frontend (ou usuário/sessão) quebrou a premissa do teste existente? (Ex: valor `null`, payload incompleto, falta de autorização/policy, eager loading ausente).

### 3. Vacinar o Teste (Red Factual)
- Adicione um caso de teste com o payload / cenário real da tela.
- Execute o teste no terminal e confirme a falha (Red).

### 4. Menor Diff Funcional na Causa Raiz
- Corrija **onde o problema é originado**, e não no sintoma.
- Não mascare o erro com `?? null` ou `isset()` no lugar onde o erro explodiu se a causa raiz for uma query ou validação anterior que não deveria ter permitido o dado nulo.
- Roda o teste no terminal (Green).
- Confirmação direta ao usuário com o diff exato e o resultado do teste.

---

## MODO 3: TRIVIAL (Ajuste Direto)

- Se a tarefa for pontual (trocar um texto, adicionar um campo óbvio, ajustar uma propriedade sem impacto colateral):
  - Não faça perguntas.
  - Não crie Micro-Spec.
  - Aplique o menor diff funcional, valide e encerre.

---

## Checklist de Qualidade de Resposta
- [ ] A resposta foi concisa, sem preâmbulos ("Com certeza!", "Entendido, vou...")?
- [ ] O comando de validação foi executado no terminal antes de declarar a tarefa pronta?
- [ ] O código tocou apenas os arquivos dentro do Blast Radius?
- [ ] O menor diff funcional venceu?
