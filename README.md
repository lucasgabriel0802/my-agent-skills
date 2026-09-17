# My Agent Skills 🚀

Repositório pessoal de **Skills** para o **Google Antigravity IDE** e agentes de IA.

Aqui ficam concentradas as automações, runbooks e instruções padronizadas para serem utilizadas em qualquer projeto ou máquina.

---

## 📂 Estrutura do Repositório

```text
my-agent-skills/
├── skills/
│   └── <nome-da-skill>/
│       ├── SKILL.md            # Instruções principais da skill
│       ├── scripts/            # (Opcional) Scripts auxiliares
│       └── references/         # (Opcional) Documentações e referências
├── .gitignore
└── README.md
```

---

## 🛠️ Skills Disponíveis

| Skill | Descrição |
|-------|-----------|
| [`commit-detalhado`](./skills/commit-detalhado/SKILL.md) | Padronização rigorosa de commits em Português seguindo *Conventional Commits* com escopo, corpo explicativo, arquivos alterados e impacto. |
| [`delphi-engenheiro`](./skills/delphi-engenheiro/SKILL.md) | Padrão de pensamento e execução para desenvolvimento, manutenção e boas práticas em Delphi (Delphi 11) e Firebird 5.0. |
| [`ponytail`](./skills/ponytail/SKILL.md) | Modo desenvolvedor sênior pragmático e eficiente. Foco em menor diff funcional, reutilização de código existente e correção na causa raiz. |

---

## 🔄 Como Sincronizar em uma Nova Máquina

Ao clonar este repositório em um novo computador com Antigravity IDE, conecte a pasta `skills` à pasta de configuração global do agente:

### No Windows (PowerShell):
```powershell
# 1. Clone o repositório
git clone https://github.com/lucasgabriel0802/my-agent-skills.git "$HOME\my-agent-skills"

# 2. Garanta que o diretório ~/.gemini/config exista
New-Item -ItemType Directory -Path "$HOME\.gemini\config" -Force

# 3. Crie a junção (link simbólico) para a pasta skills
New-Item -ItemType Junction -Path "$HOME\.gemini\config\skills" -Target "$HOME\my-agent-skills\skills" -Force
```

---

## ✍️ Como Criar uma Nova Skill

1. Crie uma pasta dentro de `skills/<nome-da-skill>` (sempre em letras minúsculas com hífen).
2. Adicione o arquivo `SKILL.md` com o frontmatter obrigatório:
   ```markdown
   ---
   name: nome-da-skill
   description: "Descreva com clareza QUANDO e PARA QUE o agente deve acionar esta skill."
   ---

   # Título da Skill

   Instruções passo a passo para o agente...
   ```
3. Salve, commite e envie para o repositório:
   ```bash
   git add .
   git commit -m "feat(skills): adicionar nova skill <nome>"
   git push origin main
   ```
