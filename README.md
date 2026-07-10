# REFINA v1.0

Orquestrador de entrega de história de usuário — valida, especifica via
OpenSpec, decide abordagem técnica com o engenheiro, delega implementação
e revisa em loop até a entrega.

## Instalar

1. Copie a pasta `.claude/agents/` para a raiz do seu repositório
   (ou só os 2 arquivos .md, se já existir a pasta).
2. Commit.
3. Abra o Claude Code no projeto e peça: "REFINA, entrega a história X"
   ou "usa o refina para essa história".

## Arquivos

- `.claude/agents/refina.md` — orquestrador
- `.claude/agents/refina-builder.md` — implementador (delegado via Task)
- `refina-documentacao.html` — documentação completa com diagrama do loop
  (abra no navegador)

## Documentação

Abra `refina-documentacao.html` no navegador para ver o diagrama do loop
e a documentação completa de gates, checklist de qualidade e o mecanismo
de detecção de padrão técnico.
