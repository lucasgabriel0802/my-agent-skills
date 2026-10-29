---
name: delphi-build-and-fix
description: Automatiza a compilação de projetos Delphi via linha de comando (MSBuild/DCC) após qualquer edição de código, analisando diagnósticos e corrigindo erros iterativamente até obter sucesso (exit code 0). Atua integrada à skill delphi-engenheiro.
triggers:
  - after_file_modification: ["*.pas", "*.dfm", "*.dpr", "*.dpk", "*.dproj", "*.inc"]
---

# Delphi Build & Auto-Fix Skill

Esta skill define as diretrizes estritas que o agente deve seguir após realizar qualquer alteração, refatoração ou criação de código em projetos Delphi (Object Pascal). Atua de forma integrada e complementar à skill **`delphi-engenheiro`**.

---

## 1. Regra de Ouro (Invariante)

> **Toda e qualquer alteração de código DEVE ser imediatamente sucedida por uma tentativa de compilação via CLI. A tarefa só é considerada concluída quando o build retornar código de saída `0` (Success).**

Não retorne ao usuário dizendo que a alteração foi concluída sem antes ter executado a compilação e garantido a ausência de erros de build.

---

## 2. Fluxo de Execução Obrigatório

Sempre que modificar um arquivo `.pas`, `.dfm`, `.dpr` ou `.dpk`:

```
[Alteração de Código (via delphi-engenheiro)]
        │
        ▼
[Executar Script / Comando de Build]
        │
   ┌────┴────────────────────────┐
   ▼                             ▼
[Build Falhou: Exit != 0]   [Build Sucesso: Exit == 0]
   │                             │
   ├─► Capturar saída (stdout/stderr)    └─► Finalizar tarefa e reportar na
   ├─► Extrair mensagens E[0-9]{4} e F[0-9]{4}  Etapa 4 do delphi-engenheiro
   ├─► Abrir arquivo(s) na linha indicada
   ├─► Aplicar correção na causa-raiz
   │
   └─► [Reexecutar Build] (repetir loop)
```

---

## 3. Comandos de Compilação Padronizados

### Opção A: Via MSBuild (Padrão para `.dproj`)
É o método recomendado, pois preserva dependências, search paths e configurações do projeto.

No Windows Terminal / Command Prompt:
```cmd
call "%BDS%\bin\rsvars.bat" && msbuild "<CAMINHO_DO_PROJETO>.dproj" /t:Build /p:Config=Debug /p:Platform=Win32 /verbosity:minimal /nologo
```

*Exemplo no Delphi 10.1 Berlin (`18.0`):*
```cmd
call "C:\Program Files (x86)\Embarcadero\Studio\18.0\bin\rsvars.bat" && msbuild "c:\DELPHI_SKILL_BERLIN\ocupacional\DPR\SkillOcp.dproj" /t:Build /p:Config=Debug /p:Platform=Win32 /verbosity:minimal /nologo
```

*Nota: Caso a variável de ambiente `%BDS%` não esteja definida globalmente, procure o script em:*
- `C:\Program Files (x86)\Embarcadero\Studio\<VERSAO>\bin\rsvars.bat` (ex: `18.0` para 10.1 Berlin, `22.0` para 11 Alexandria, `23.0` para 12 Athens).

### Opção B: Via Compilador Direto (`dcc32` / `dcc64`)
Utilizado para builds rápidos de arquivos `.dpr` com arquivos `.cfg` ou parâmetros de search path explícitos:
```cmd
call "%BDS%\bin\rsvars.bat" && dcc32 -B -Q -W "<CAMINHO_DO_PROJETO>.dpr"
```

---

## 4. Diagnóstico e Resolução de Erros Comuns

Ao analisar as falhas de build, utilize esta matriz de resolução rápida:

| Código de Erro | Significado | Ação Imediata do Agente |
| :--- | :--- | :--- |
| **`[dcc32 Error] E2003 Undeclared identifier: 'X'`** | Identificador não declarado ou unit ausente | 1. Verificar se a unit que contém o tipo/função está na seção `uses` da interface ou implementation.<br>2. Checar grafia de variáveis e métodos. |
| **`[dcc32 Error] E2010 Incompatible types: 'X' and 'Y'`** | Tipos incompatíveis na atribuição/passagem | 1. Verificar conversões explícitas necessárias (`IntToStr`, `StrToFloat`, typecasting seguro via `as` ou `is`).<br>2. Conferir assinaturas de métodos. |
| **`[dcc32 Error] E2065 Unsatisfied forward or external declaration: 'X'`** | Método declarado na classe mas não implementado | Adicionar o bloco de implementação correspondente `procedure/function TClasse.Metodo(...)` na seção `implementation`. |
| **`[dcc32 Fatal] F1026 File not found: 'X.dcu'`** | Unit ou dependência não encontrada no Search Path | 1. Verificar se o arquivo `.pas` existe no projeto.<br>2. Adicionar o caminho da pasta aos Search Paths do `.dproj` ou na flag `-U` do DCC. |
| **`[dcc32 Error] E1030 Invalid compiler directive: 'X'`** | Diretiva de compilação com sintaxe errada | Revisar blocos `{$IFDEF}`, `{$ENDIF}` e sintaxe de diretivas. |

---

## 5. Critérios de Conclusão da Iteração

O ciclo de auto-fix encerra-se apenas sob as seguintes condições:

1. **Sucesso Total:** O comando de build retorna código `0` com zero `[Fatal]` e zero `[Error]`.
2. **Tratamento de Warnings e Hints:** Warnings críticos (`W1000` em diante) devem ser revisados para evitar memory leaks (ex: variáveis locais não inicializadas ou objetos criados sem `Free/FreeAndNil`).
3. **Resumo das Correções:** Ao responder ao usuário (integrado à confirmação final da `delphi-engenheiro`), liste:
   - Os arquivos modificados originalmente.
   - Os erros de compilação detectados durante as passagens do build.
   - O diff/resumo das correções aplicadas para resolver os erros.
   - A confirmação de que o build final foi validado com sucesso (exit code 0).

